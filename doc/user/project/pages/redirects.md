---
stage: Release
group: Release Management
info: To determine the technical writer assigned to the Stage/Group associated with this page, see https://about.gitlab.com/handbook/engineering/ux/technical-writing/#designated-technical-writers
---

# Create redirects for GitLab Pages

> - [Introduced](https://gitlab.com/gitlab-org/gitlab-pages/-/issues/24) in GitLab Pages 1.25.0 and GitLab 13.4.
> - It's [deployed behind a feature flag](#enable-or-disable-redirects), disabled by default.
> - To use it in GitLab self-managed instances, ask a GitLab administrator to [enable it](#enable-or-disable-redirects).

CAUTION: **Warning:**
This feature might not be available to you. Check the **version history** note above for details.

In GitLab Pages, you can [enable](#enable-or-disable-redirects) the redirects feature to configure rules to forward one URL to another using HTTP redirects. GitLab Pages uses
[Netlify style redirects](https://docs.netlify.com/routing/redirects/#syntax-for-the-redirects-file).

## Supported features

GitLab Pages only supports the
[`_redirects` plain text file syntax](https://docs.netlify.com/routing/redirects/#syntax-for-the-redirects-file),
and `.toml` files are not supported.

Redirects are only supported at a basic level, and GitLab Pages doesn't support all
[special options offered by Netlify](https://docs.netlify.com/routing/redirects/redirect-options/):

| Feature | Supported | Example |
| ------- | --------- | ------- |
| Redirects (`301`, `302`) | **{check-circle}** Yes | `/wardrobe.html /narnia.html 302`
| Rewrites (other status codes) | **{dotted-circle}** No | `/en/* /en/404.html 404` |
| [Splats](https://docs.netlify.com/routing/redirects/redirect-options/#splats) | **{dotted-circle}** No | `/news/*  /blog/:splat` |
| Placeholders | **{dotted-circle}** No | `/news/:year/:month/:date/:slug  /blog/:year/:month/:date/:slug` |
| Query parameters | **{dotted-circle}** No | `/store id=:id  /blog/:id  301` |
| Force ([shadowing](https://docs.netlify.com/routing/redirects/rewrites-proxies/#shadowing)) | **{dotted-circle}** No | `/app/  /app/index.html  200!` |
| Domain-level redirects | **{dotted-circle}** No | `http://blog.example.com/* https://www.example.com/blog/:splat 301` |
| Redirect by country or language | **{dotted-circle}** No | `/  /anz     302  Country=au,nz` |
| Redirect by role | **{dotted-circle}** No | `/admin/*  200!  Role=admin` |

NOTE: **Note:**
Supported paths must start with a forward slash `/`.

## Create redirects

To create redirects after [enabling](#enable-or-disable-redirects) the feature,
create a configuration file named `_redirects` in the `public/` directory of your
GitLab Pages site.

If your GitLab Pages site uses the default domain name (such as
`namespace.gitlab.io/projectname`) you must prefix every rule with the project name:

```plaintext
/projectname/redirect-portal.html /projectname/magic-land.html 301
/projectname/cake-portal.html /projectname/still-alive.html 302
/projectname/wardrobe.html /projectname/narnia.html 302
/projectname/pit.html /projectname/spikes.html 302
```

If your GitLab Pages site uses [custom domains](custom_domains_ssl_tls_certification/index.md),
no project name prefix is needed. For example, if your custom domain is `example.com`,
your `_redirect` file would look like:

```plaintext
/redirect-portal.html /magic-land.html 301
/cake-portal.html /still-alive.html 302
/wardrobe.html /narnia.html 302
/pit.html /spikes.html 302
```

## Files override redirects

Files take priority over redirects. If a file exists on disk, GitLab Pages serves
the file instead of your redirect. For example, if the files `hello.html` and
`world.html` exist, and the `_redirects` file contains the following line, the redirect
is ignored because `hello.html` exists:

```plaintext
/projectname/hello.html /projectname/world.html 302
```

NOTE: **Note:**
GitLab does not support Netlify's
[force option](https://docs.netlify.com/routing/redirects/rewrites-proxies/#shadowing)
to change this behavior.

## Debug redirect rules

If a redirect isn't working as expected, or you want to check your redirect syntax, visit
`https://[namespace.gitlab.io]/projectname/_redirects`, replacing `[namespace.gitlab.io]` with
your domain name. The `_redirects` file isn't served directly, but your browser
displays a numbered list of your redirect rules, and whether the rule is valid or invalid:

```plaintext
11 rules
rule 1: valid
rule 2: valid
rule 3: error: splats are not supported
rule 4: valid
rule 5: error: placeholders are not supported
rule 6: valid
rule 7: error: no domain-level redirects to outside sites
rule 8: error: url path must start with forward slash /
rule 9: error: no domain-level redirects to outside sites
rule 10: valid
rule 11: valid
```

## Enable or disable redirects

Redirects in GitLab Pages is under development and not ready for production use. It is
deployed behind a feature flag that is **disabled by default**.

For [Omnibus installations](../../../administration/pages/index.md), define the
`FF_ENABLE_REDIRECTS` environment variable in the
[global settings](../../../administration/pages/index.md#global-settings).
Add the following line to `/etc/gitlab/gitlab.rb` and
[reconfigure the instance](../../../administration/restart_gitlab.md#omnibus-gitlab-reconfigure).

```ruby
gitlab_pages['env']['FF_ENABLE_REDIRECTS'] = 'true'
```

For [source installations](../../../administration/pages/source.md), define the
`FF_ENABLE_REDIRECTS` environment variable, then
[restart GitLab](../../../administration/restart_gitlab.md#installations-from-source):

```shell
export FF_ENABLE_REDIRECTS="true"
/path/to/pages/bin/gitlab-pages -config gitlab-pages.conf
```
