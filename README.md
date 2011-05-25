## SUMMARY

File storage backend `django_dust` (Distributed Upload STorage) can store
files coming into a Django site on several servers simultaneously, using
WebDAV.

This works for files uploaded by users through the admin or through custom
Django forms, and also for files created by the application code, provided it
uses the standard [storage API][1].

This backend is useful for sites deployed in a multi-server environment, in
order to accept uploaded files and have them available on all media servers
immediately for subsequent web requests that could be routed to any machine.

## USE CASES

### Preliminary warning

Django's built-in `FileSystemStorage` goes to great lengths to avoid race
conditions. It is difficult for `django_dust`'s `DistributedStorage` to
provide the same guarantees, because of the [CAP theorem][2]. For this reason,
`django_dust` uses a master copy and is designed to obtain eventual
consistency on the media servers.

### Recommended setup

In an infrastructure for a Django website, each server has one (or several)
of the following roles:

- Frontend servers handle directly HTTP requests for media and static files,
  and forward other requests to the **application servers**.
  For `django_dust`, the interesting part is that they are serving media
  files, so we call them **media servers**.
- Application servers run the Django application. This is where
  `django_dust` is installed.
- Database servers support the database.

If you have several application servers, you should store a master copy of
your media files on a NAS or a SAN attached to all your application servers.
If you have a single application server, you can also store the master copy
on the application server itself.
In both cases, `django_dust` replicates uploaded files on all the media servers.

For the media servers, serving the files from the local filesystem is more efficient than serving them from a NAS or a SAN. Also, it means that they
don't depend on the NAS or SAN.

### Keeping media directories synchronized

If you bring an additional media server online, you must synchronize the
content of `MEDIA_ROOT` from the master copy, for example with `rsync`.

If a media server is unavailable, `django_dust` will log a message at level
`ERROR` for each failed upload. Once the server is back online, you can
re-synchronize the contents of `MEDIA_ROOT` from the master copy with `rsync`.
You can also set up a cron if you get random failures, for instance during
load peaks.

`django_dust` used to keep a queue of failed operations to repeat them
afterwards. This is inherently prone to data loss, because the order of PUT
and DELETE operations matters, and retrying failed operations later breaks
the order. So, use `rsync` instead, it's fast enough.

### Low concurrency situations

You may have several servers for high availability, but still expect a low
concurrency on write operations. This is a common pattern for editorial
websites. In such circumstances, you can decide not to store a master copy of
your media files on the application server. See `DUST_USE_LOCAL_FS` below.

This trade-off has two consequences:

- `django_dust` will raise an error if a file can not be uploaded to all media
  servers. You no longer have high availability for write operations.
- Race conditions become possible: if two people upload different files with
  the same name at the same time, you may randomly end up with one file or the
  other on each media server.

## SETTINGS

### `DUST_MEDIA_HOSTS`

Default: `()`

List of host names for the media servers.

The WebDAV URL for a given media file is the same as its HTTP URLs, built
using `MEDIA_URL`, except that the host name changes. It is not possible to
use HTTPS.

### `DUST_USE_LOCAL_FS`

Default: `True`

When `True`, `django_dust` will run all file storage operations on
`MEDIA_ROOT` first, then replicate them to the media servers.

When `False`, `django_dust` will only store the files on the media servers.
See "Low concurrency situations" above.

### `DUST_TIMEOUT`

Default: `2`

Timeout in seconds for WebDAV operations.

This controls the maximum amount of time an upload operation can take. Note
that all uploads run in parallel.


## INSTALLATION AND SETUP

1.  Download and install the package from PyPI:

        $ pip install django_dust

    `django_dust` requires [httplib2][3] >= 0.4. (TODO: use urllib2.)

2.  Add django_dust to `INSTALLED_APPS`:

        INSTALLED_APPS += 'django_dust',

3.  Set `django_dust` as a default file backend, if you want all your models
    to use it:

        DEFAULT_FILE_STORAGE = 'django_dust.storage.DistributedStorage'

    This is optional. You can also enable this backend only for some
    fields in your models.

4.  Define the list of your media servers:

        DUST_MEDIA_HOSTS = ['media-%02d:8080' % i for i in range(12)]

    OK, maybe you don't have 12 servers just yet.

5.  Make sure you have configured `MEDIA_ROOT` and `MEDIA_URL`.

6.  Set up WebDAV on your media servers.

## SETTING UP WEBDAV ON THE MEDIA SERVERS

The backend uses WebDAV to transfer files to media servers. It requires that
those servers support the PUT and DELETE (TODO: and MKCOL?). Despite of them
being part of HTTP itself they are traditionally implemented by external
modules supporting WebDAV. Hence make sure that  web servers have this module
enabled. For security reasons it's critical to enable it only for internal IPs
to prevent external users from being able to write and delete server files.

An example of lighttpd config:

    server.modules += (
      "mod_webdav",
    )

    $HTTP["remoteip"] ~= "^192\.168\.0\.[0-9]+$" {
      "webdav.activate = "enable"
    }

An example of nginx config, assuming it was compiled `--with-http_dav_module`:

    server {
        listen 192.168.0.10;
        location / {
            root /var/www/media;
            dav_methods PUT DELETE;
            create_full_put_path on;
            dav_access user:rw group:r all:r;
            allow 192.168.0.1/24;
            deny all;
        }
    }


[1]: http://docs.djangoproject.com/en/dev/ref/files/storage/
[2]: http://en.wikipedia.org/wiki/CAP_theorem
[3]: http://code.google.com/p/httplib2/