If you’re running rsyslog and want to run it securely, you’re going to want to disable SSLv2 and SSLv3.  If you’re running rsyslog in a PCI environment you’ll HAVE to.  Given recent PCI standards you’ll also have to disable TLS 1.0 (and likely 1.1 in the near future).

First thing you’ll need to do is familiarize yourself with GnuTLS priority strings, and what string fits your needs.  Go here: GnuTLS Priority Strings for a full read on priority strings.

For example purposes here I’m using:

    SECURE192:-VERS-ALL:+VERS-TLS1.2

Which enables all 192-bit ciphers, and disables all versions of TLS except 1.2.

If you’re using imrelp, you’re in luck.  The imrelp input has a `tls.prioritystring` setting so just set it to whatever fits your needs:

```
input(
       type = "imrelp"
       port = 2514
       ruleset = "relp_messages"
       tls = "on"
       tls.prioritystring = "SECURE192:-VERS-ALL:+VERS-TLS1.2"
       tls.authMode = "name"
       tls.permittedPeer = [ "*.mycompany.com", "*.int.mycompany.com"]
)
```

You’re done.

Sadly, imtcp does not include any such setting.  To enforce a priority string on imtcp you’ll have to set it at what rsyslog calls the StreamDriver level, namely gtls.  In order to do this you will to build GnuTLS from source with the following flags passed to the configure script:

    --disable-ssl3-support --disable-ssl2-support --with-default-priority-string="SECURE192:-VERS-ALL:+VERS-TLS1.2"

Then do the usual make, make install and the new gnutls libs will operate based on the priority string you specified.  You’ll have to install these updated librarys to all rsyslog hosts you want to lock down.  Or if you’re like me and need to keep things compartmentalized for easy distribution and scaling, you can just add all `/usr/lib64/libgnutls*` libraries to your rsyslog package and update your `LD_LIBRARY_PATH` so that they’re loaded first.

If you use both secure imtcp and imrelp, the good news is that this method of recompiling GnuTLS will apply to both.  There’s no need to specify a `tls.prioritystring` in your imrelp input.

Once you’ve updated and restarted rsyslog you can verify everything is working using `openssl s_client`:

   openssl s_client -connect localhost:2514 -tls1_1

If your priority string is set to enable TLS1.1 it should handshake and you should get a server certificate back.  If not, you’ll get back a handshake error.

To test against other protocols / versions:

- `ssl2` - just use SSLv2
- `ssl3` - just use SSLv3
- `tls1_2` - just use TLSv1.2
- `tls1_1` - just use TLSv1.1
- `tls1` - just use TLSv1.0

If you get handshake errors on versions you’d expect to connect on, it’s likely a bad priority string setting.  Unfortunately when you compile GnuTLS there’s no test run that determines if your priority string is formatted correctly and is valid.  There’s simply too many options and permutations to a priority string.  So you may have to go through the compilation processes several times before you get it right.  Once you get a priority string that works, do yourself a favor and note it down somewhere.  It will come up again.
