# GoDaddy DNS provider for Terraform

This plug-in enables the management of individual DNS resource records for domains hosted on GoDaddy DNS servers.

It only manages DNS (no e.g. domain management) and aims to manage individual DNS resource records (not the whole domain), while preserving existent records and tolerating external modifications.

## GoDaddy API access restrictions

Unfortunately, at the start of May 2024 GoDaddy suddenly decided to restrict access to their DNS management API. There were no announcements of this policy change and as of 2024.05.08 it is still not reflected in [developers' documentation](https://developer.godaddy.com/getstarted), but there are [many](https://www.reddit.com/r/godaddy/comments/1chs1j8/godaddy_access_denied_via_apicall/) [reports](https://www.reddit.com/r/godaddy/comments/1bl0f5r/am_i_the_only_one_who_cant_use_the_api/) of the same [problem](https://community.letsencrypt.org/t/getting-unauthorized-url-error-while-trying-to-get-cert-for-subdomains/217329) (i.e. getting an "Authenticated user is not allowed access" error while accessing DNS API with the valid keys). So for now the development of this provider is stopped.

## Usage

Example usage (set credentials in `GODADDY_API_KEY` and `GODADDY_API_SECRET` env vars):

``` terraform
terraform {
  required_providers {
    godaddy-dns = {
      source = "registry.terraform.io/veksh/godaddy-dns"
    }
  }
}

# keys usually set with env GODADDY_API_KEY and GODADDY_API_SECRET
provider "godaddy-dns" {}

# to import existing records:
# terraform import godaddy-dns_record.new-cname domain.com:CNAME:alias:testing.com
resource "godaddy-dns_record" "new-cname" {
  domain = "domain.com"
  type   = "CNAME"
  name   = "alias"
  data   = "test.com"
}
```

## Supported RR types, mode of operation

It currently supports `A`, `AAAA`, `CNAME`, `MX`, `NS` and `TXT` records. `SRV` are not supported; if anyone hosting AD on GoDaddy or uses them for VOIP or something like that, please let me know by creating an issue.

GoDaddy API does not have stable identities for DNS records, and in case of external modifications (e.g. via web console) behaviour is slightly different for "single-valued" vs "multi-valued" records
- for "single-valued" record types (`CNAME`) there could be only 1 record of this type with a given name, so these are just replaced by update
- for "multi-valued" record types (`A`, `MX`, `NS`, `TXT`) there could be several records
with a given name (e.g. multiple MXes with different priorities and targets), so matching is done on value
- if record's value is modified outside of Terraform, it is treated as a completely different record and is preserved, while original record is considered gone and is re-created on `apply` (use `refresh` + `import` to re-link modified record back to original).

## Differences vs alternative providers

Differences vs n3integration provider and its forks:
- configuration object is record, not (top-level) domain no need to bring it all under terraform management
- could safely manage a subset of records (e.g. only one TXT or CNAME at domain top), keeping the rest intact
- updates do not result in scary plans threatening to destroy all the records in the whole domain
- `destroy` is fully supported and removes only previously created records
