+++
title = "Testing an ansible collection"
date = 2020-05-08
tags = ["ansible"]
draft = false
+++

When wanting to work on a collection for `ansible` it is (for now[^fn:1]) important
to check it out in a very specific folder structure or it will not be possible
to run the tests for it.

When trying to run `ansible-test integration` it will otherwise throw the
following error:

```text
ERROR: The current working directory must be at or below:

 - an Ansible collection: {...}/ansible_collections/{namespace}/{collection}/

Current working directory: <some_other_dir>
```

The collection must be really placed inside a `subfolder` of a folder called
`ansible_collections` and neither `namespace` nor `collection` can contain
any symbols except `alphanumberics` or `underscores` (`[a-zA-Z0-9_]`):

```sh
ansible_collections
└── namespace
    └── collection
```

So if you want to work on an upstream collection (e.g. [community.general](https://github.com/ansible-collections/community.general)) you
should create an intermediate folder `community` and clone the collection into
the `general` folder (contrary to the default checkout which would be
`community.general`):

```sh
ansible_collections
└── community
    └── general
```

Inside `general` you can now use run `ansible-test integration` to run
the integration tests successfully:

```sh
cd ansible_collections/community/general
poetry init --name community.general --dependency=ansible --dependency=pyyaml --dependency=jinja2 -n
poetry install
poetry run ansible-test integration
```

[^fn:1]: This might hopefully become easier in the future: <https://github.com/ansible/ansible/issues/60215>
