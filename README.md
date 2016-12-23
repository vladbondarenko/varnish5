# varnish5

Added compiled varnish-5.0.0 + libvmod-geoip2 installed + maxmind DB

Added modules: directors,header,saintmode,tcp,vsthrottle,xkey,var,cookie

vcl 4.0;
----------

cookie:
import cookie;

    sub vcl_recv {
        if (req.http.cookie) {
            cookie.parse(req.http.cookie);
            # Either delete the ones you want to get rid of:
            cookie.delete("cookie2");
            # or filter everything but a few:
            cookie.filter_except("SESSIONID,PHPSESSID");

            # Store it back into req so it will be passed to the backend.
            set req.http.cookie = cookie.get_string();

            # If empty, unset so the builtin VCL can consider it for caching.
            if (req.http.cookie == "") {
                unset req.http.cookie;
            }
        }
    }

----------

header:
import header;

    sub vcl_backend_response {
        if (beresp.http.Set-Cookie) {
            # Add another line of Set-Cookie in the response.
            header.append(beresp.http.Set-Cookie, "VSESS=abbabeef");

            # CMS always set this, but doesn't really need it.
            header.remove(beresp.http.Set-Cookie, "JSESSIONID=");
        }
    }

----------

saintmode+directors:
import saintmode;
import directors;

    backend tile1 { .host = "192.0.2.11"; .port = "80"; }
    backend tile2 { .host = "192.0.2.12"; .port = "80"; }

    sub vcl_init {
        # Instantiate sm1, sm2 for backends tile1, tile2
        # with 10 blacklisted objects as the threshold for marking the
        # whole backend sick.
        new sm1 = saintmode.saintmode(tile1, 10);
        new sm2 = saintmode.saintmode(tile2, 10);

        # Add both to a director. Use sm0, sm1 in place of tile1, tile2.
        # Other director types can be used in place of random.
        new imagedirector = directors.random();
        imagedirector.add_backend(sm1.backend(), 1);
        imagedirector.add_backend(sm2.backend(), 1);
    }

    sub vcl_backend_fetch {
        # Get a backend from the director.
        # When returning a backend, the director will only return backends
        # saintmode says are healthy.
        set bereq.backend = imagedirector.backend();
    }

    sub vcl_backend_response {
        if (beresp.status >= 500) {
            # This marks the backend as sick for this specific
            # object for the next 20s.
            saintmode.blacklist(20s);
            # Retry the request. This will result in a different backend
            # being used.
            return (retry);
        }
    }

----------

tcp:
import tcp;

    sub vcl_recv {
        # Shape (pace) the data rate to avoid filling router buffers for a
        # single client.
        if (req.url ~ ".mp4$") {
                tcp.set_socket_pace(1000);   # KB/s.
        }

        # We want to change the congestion control algorithm.
        if (tcp.get_estimated_rtt() > 300) {
            set req.http.x-tcp = tcp.congestion_algorithm("hybla");
        }
    }

----------

var:
import var;

    sub vcl_recv {
        # Set and get some values.
        var.set("foo", "bar");
        set req.http.x-foo = var.get("foo");

        var.set_int("ten", 10);
        var.set_int("five", 5);
        set req.http.twenty = var.get_int("ten") + var.get_int("five") + 5;

        # VCL will use the first token to decide final data type. Headers are strings.
        # set req.http.X-lifetime = var.get_int("ten") + " seconds"; #  Won't work.
        set req.http.X-lifetime = "" + var.get_int("ten") + " seconds";  # Works!

        var.set_duration("timedelta", 1m);  # 60s
        set req.http.d1 = var.get_duration("timedelta");

        var.set_ip("endpoint", client.ip);
        set req.http.x-client = var.get_ip("endpoint");

        # Unset all non-global variables.
        var.clear();

        # Demonstrate use of global variables as state flags.
        if (req.url ~ "/close$") {
            var.global_set("open", "no");
        }
        else if (req.url ~ "/open$") {
            var.global_set("open", "yes");
        }

        if (var.global_get("open") != "yes") {
            return (synth(200, "We are currently closed, sorry!"));
        }
    }

----------

vsthrottle:
import vsthrottle;

    sub vcl_recv {
        # Varnish will set client.identity for you based on client IP.

        if (vsthrottle.is_denied(client.identity, 15, 10s)) {
            # Client has exceeded 15 reqs per 10s
            return (synth(429, "Too Many Requests"));
        }

        # There is a quota per API key that must be fulfilled.
        if (vsthrottle.is_denied("apikey:" + req.http.Key, 30, 60s)) {
                return (synth(429, "Too Many Requests"));
        }

        # Only allow a few POST/PUTs per client.
        if (req.method == "POST" || req.method == "PUT") {
            if (vsthrottle.is_denied("rw" + client.identity, 2, 10s)) {
                return (synth(429, "Too Many Requests"));
            }
        }
    }

----------

xkey:
import xkey;

    acl purgers {
        "203.0.113.0"/24;
    }

    sub vcl_recv {
        if (req.method == "PURGE") {
            if (client.ip !~ purgers) {
                return (synth(403, "Forbidden"));
            }
            set req.http.n-gone = xkey.purge(req.http.key);
            # or: set req.http.n-gone = xkey.softpurge(req.http.key)

            return (synth(200, "Invalidated "+req.http.n-gone+" objects"));
        }
    }
