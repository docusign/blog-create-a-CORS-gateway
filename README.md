# Creating Apache CORS Gateways for DocuSign

This repository provides step by step instructions for
creating private CORS gateways for use with
DocuSign.

This repository accompanies the DocuSign Developer blog post
[Building Single-Page Applications with DocuSign and CORS: Part 2](https://www.docusign.com/blog/dsdev-building-single-page-applications-with-docusign-and-cors-part-2/).

## Summary
You'll set up two CORS proxies/gateways, one for demo.docusign.net, and one
for account-d.docusign.com. The account-d server is used to lookup
the user's account(s) and other details
via the
[userinfo](https://developers.docusign.com/esign-rest-api/guides/authentication/user-info-endpoints)
method after they login. The OAuth Implicit Grant flow itself uses redirects, CORS is not needed.

 You’ll create two HTTPS CORS gateways:
 * `https://cors-a.worldwidecorp.us` will forward to `https://account-d.docusign.com`.
   As noted above, this gateway will only be used for the userinfo API method.
 * `https://cors-d.worldwidecorp.us` will forward to `https://demo.docusign.net`

Both will use port 1443. You can use a different port if you'd like.

In these examples, the domain name `worldwidecorp.us` is used;
you will substitute your server's dns address.
It can include subdomain names. Eg `cors-d.my-sub-domain.mycompany.com`

Both proxies will run on one instance of Apache as two virtual hosts.
The proxies use port 1443 and SSL/TLS.
The proxies can be installed behind your firewall,
on your private network,
or on the public Internet but they do need to be reachable by the
browsers that will be running your web application.

## Audience
These instructions are written for an experienced system and network
administrator. If you have questions about the
instructions, please consult [ServerFault](https://serverfault.com/).

## Information Security
A CORS gateway is *literally* a
[Man in the Middle](https://en.wikipedia.org/wiki/Man-in-the-middle_attack) machine.
It **must** be properly configured and monitored to
prevent a loss of **all** of your application's data.

You **must** consult with your Information Security
department or consultant to create a secure gateway.
These instructions do **not** provide information
on Information Security-related issues or settings.

See the [LICENSE](LICENSE) file for additional information.

## Installation

#### 1. Set up the two host names in the DNS system to enable access by the clients.

Eg `cors-d.mycompany.com` and `cors-a.mycompany.com`.
Both addresses need their DNS records to resolve to your Apache server.

#### 2. Set up Apache on your server.
These examples have been tested on a Linux host. Use similar steps for Windows OS.

1. Install Apache version 2.4 (or later) on your server. It may already be installed.
Consult the instructions for your operating system for specific
installation steps. You do not need to install PHP or other
language support for the CORS gateways.

1. Enable Apache to restart when the system is rebooted. The commands
   for doing this vary depending on your operating system.
   Reboot your server and check that Apache was restarted automatically.

1. Enable additional modules:

   `a2enmod ssl rewrite headers proxy_http setenvif cache`

   Depending on your operating system, you may need to run
   the command as root or as the administrator.

1. Restart Apache. The command for restarting Apache will vary
   depending on your operating system. For many Linux systems the
   restart command is

   `sudo apachectl start`

#### 3. Add the HTTP virtual host configuration files
The HTTP virtual host files will only be used by the Let's Encrypt software
to create the SSL hosts.

1. Copy files `100-cors_a.conf` and `110-cors_d.conf`
   from this repository to the `apache2/sites-available` configuration
   directory. Note: the configuration files are named
   **...cors_a** and **...cors_b** (with underscore characters)
   due to Apache restrictions for configuration file names. The
   websites are **cors-a** and **cors-b**.
1. Update the `ServerName` directives in both files.
1. Create the webroot directories listed in the two files:
   * `/var/www/cors-a`
   * `/var/www/cors-d`

   Ensure that the Apache `user` and/or `group` has access to the
   directories. The Apache user and group are listed in the
   main Apache configuration file, `apache2.conf`. If environment
   variables are used, check the file `envvars` in the Apache2
   configuration directory.

1. Enable the sites on your server:
   * `sudo a2ensite 100-cors_a.conf`
   * `sudo a2ensite 110-cors_d.conf`

#### 4. Enable the SSL hosts
These steps use free SSL certificates from [Let's Encrypt](https://letsencrypt.org/)
for the SSL sites. If your gateways are only on your private network
then use an appropriate SSL/TLS certificate.

To install a Let's Encrypt certificate on Linux, use the free
[certbot](https://certbot.eff.org/)
program. You'll need your server's Linux distribution name and
version. See the  [instructions](https://www.cyberciti.biz/faq/find-linux-distribution-name-version-number/)
for determining the information.

If you receive the error `E: The value 'stretch-backports' is invalid for APT::Default-Release as such a release is not available in the sources`
then you need to add the correct backports to your source list.
See the backports [instructions](https://backports.debian.org/Instructions/#index2h2).

Use the [instructions](https://certbot.eff.org/)
to run the certbot program in apache2 mode.

Choose the `No redirect` option since you'll be disabling the HTTP sites.

Now examine your Apache2 `sites-available` directory. You should see two
new configuration files, created by the certbot utility:

* 100-cors_a-le-ssl.conf
* 110-cors_d-le-ssl.conf

"le" stands for Let's Encrypt. If you check the `sites-enabled` directory,
you should see all four sites ready to go:

* 100-cors_a.conf
* 100-cors_a-le-ssl.conf
* 110-cors_d.conf
* 110-cors_d-le-ssl.conf

#### 5. Update site configurations

1. Disable the HTTP sites:
   * `sudo a2dissite 100-cors_a.conf`
   * `sudo a2dissite 110-cors_d.conf`

2. Edit the `100-cors_a-le-ssl.conf` configuration file in the `sites-available` directory,
   * Update steps 1-4 as shown within the file.
   * Step 1: The Let's Encrypt app overwrites the port number in
     the `VirtualHost` statement. You will need to update it to
     the port number you want.
   * Step 4: Set the **Access-Control-Allow-Origin** setting) to
     **exactly** match the *scheme* (http or https), the hostname, and the
     port of your web app's html page origin,
     or the CORS request will fail. You can't use a wildcard here.
3. Repeat for the `110-cors_d-le-ssl.conf` configuration file.

#### 6. Add Listen directive
To enable port 1443, you will add a [Listen](https://httpd.apache.org/docs/2.4/bind.html)
directive:

**Listen 1443**

The directive can only be used once within the Apache configuration files.
You can add it to the master httpd.conf file, or, preferably, add
it to the **ports.conf** file in the **apache2/** configuration directory.

Add the the **Listen 1443** line after the existing **Listen 80** line.

#### 7. Test your gateways, part I

1. Restart Apache to use the new host configurations:

   `sudo apachectl restart`

    Note: The command for restarting Apache may be different on your system.

    The start and restart commands for Apache include extensive
    error checking of the configuration files.

1. Test the `account-d.docusign.com` proxy with a
   [curl](https://curl.haxx.se/) command from a different computer:

   `curl https://cors-a.worldwidecorp.us:1443/oauth/userinfo`

   **Note:** Remember to use your own gateway hostname and the right port number.

   You **should** receive an error response since we did not
   include an `Authorization` header. You should see a similar
   error if you directly contact the `account-d.docusign.com` service:

   `curl https://account-d.docusign.com/oauth/userinfo`

1. Test the `demo.docusign.net` proxy with this
    [curl](https://curl.haxx.se/) command from a different computer:

    `curl https://cors-d.worldwidecorp.us:1443/restapi/v2/accounts/1/envelopes/2`

    **Notes:** Remember to use your own hostname and the right port number.

    You **should** receive an error response since we did not
    include an `Authorization` header. You should see a similar
    error if you directly contact the `demo.docusign.net` service:

    `curl https://demo.docusign.net/restapi/v2/accounts/1/envelopes/2`

#### 8. Test your gateways, part II

You're now ready to test your web page application's access
to DocuSign via your new gateways. Remember that your
JavaScript program must use the CORS option to communicate
via the gateways.

#### 9. Production
For production, you'll need additional gateways, one for
`account.docusign.com`, and one for each of the production
platforms you need to use (`www`, `na2`, `na3`, `eu`, etc).

#### Problems and Solutions
* When you restarted Apache, you should not see any error messages.
Researching the error messages on the web will often provide solutions.

* Check your Apache access and error logs for more information if the sites
aren't responding.

* Apache configurations often have permissions issues. There are many
resources available on the internet to assist you with permissions problems.
Remember to check permissions for the parent directories of your document
directories too.

* Confirm that the Apache main configuration file is enabling virtual hosts.

* If the host name is not found then check your dns entries, and use ping to
confirm that basic name lookup and networking is okay.

* If the CORS request doesn't work:
  CORS is implemented by the browser, not by the fetch or jQuery AJAX methods.

  Check the browser's debugger for additional information on the problem.
  For example, the Chrome debugger provided this error message in the debugger's console:

> Failed to load https://cors-d.example.com/oauth/user_info: Response to preflight request doesn't pass access control check: The 'Access-Control-Allow-Origin' header has a value 'https://localhost' that is not equal to the supplied origin. Origin 'http://localhost' is therefore not allowed access. Have the server send the header with a valid value, or, if an opaque response serves your needs, set the request's mode to 'no-cors' to fetch the resource with CORS disabled.

Solution: update the `Access-Control-Allow-Origin` header value to
**exactly** match the host value.
In this case, changing `https` to `http` solved the problem.

##### Which HTML files can use the CORS gateways?

The example proxy host files only allow browsers
to use the gateway if the initial HTML file was loaded from
a specific server:

````
#### Update to the hostname that hosts your webapp
Header always set Access-Control-Allow-Origin "https://your_apps_host.com"
````
(Lines 30-31)

You cannot create a wildcard CORS gateway since the
CORS standard does not allow wildcard client support
if the `Authentication` header will be proxied.

[Documentation](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS#Access-Control-Allow-Credentials)
for the Access-Control-Allow-Credentials option.
