---
page_title: "Adding images to your registry"
subcategory: ""
description: |-
  Sync images from the Chainguard catalog into your organization's registry
  with the chainguard_image_repo resource.
---

# Adding images to your registry

Use the [`chainguard_image_repo`](../resources/image_repo.md) resource to
create container image repositories in your organization's registry. Each repo
you declare pulls from a catalog source you name, so Terraform becomes the
record of which container images your organization has added to its registry.

This guide assumes you are already authenticated. Refer to
[Authenticating the Chainguard provider](authentication.md) if you are not.

## Look up your group

A repo belongs to a group, identified by its UIDP. Look up your organization's
root group by name so you don't have to hardcode the ID:

```terraform
data "chainguard_group" "group" {
  name = "YOUR.ORG"
}
```

Replace `YOUR.ORG` with your verified organization name.

## Create and sync repositories

Declare one repo per container image you want to add. Passing the list of image
names to the `for_each` meta-argument keeps the configuration compact and lets
you add or remove images by editing a single list. The `sync_config` block
points each repo at its catalog source:

```terraform
resource "chainguard_image_repo" "repo" {
  for_each = toset(
    [
      "nginx-fips",
      "python-fips",
      "go-fips",
    ]
  )

  parent_id = data.chainguard_group.group.id
  name      = each.key

  sync_config {
    source = each.key
  }

  lifecycle {
    ignore_changes = [
      active_tags,
      bundles,
      tier,
      readme,
      description,
    ]
  }
}
```

Here `name` and `source` share the same value (`each.key`), so each repo in
your registry matches its catalog name. Set them separately if you want a local
repo name that differs from the source.

### Why ignore certain attributes

The `ignore_changes` block tells Terraform to disregard server-side changes to
attributes the platform manages for synced repos. `active_tags`, `bundles`,
and `tier` reflect catalog state that the sync process updates, and `readme`
and `description` are populated from the catalog. Because these values are set
by the API rather than your configuration, Terraform would otherwise report
them as drift on every plan and attempt to reset them to your configured values
on the next apply.

## Add or remove images

To add another container image, add its name to the `for_each` list and run
`terraform apply`. To stop syncing one, remove it from the list.

~> **Note:** Deleting a `chainguard_image_repo` is a no-op on the Chainguard
side — the resource is removed from Terraform state, but the image repo itself
is not deleted. Remove repos through your normal registry administration
process.

## Next steps

- Grant access to the repos with
  [`chainguard_rolebinding`](../resources/rolebinding.md).
- Inspect a repo's available tags with the
  [`chainguard_image_repo`](../data-sources/image_repo.md) data source.
