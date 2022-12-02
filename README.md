# Setup MAAS GitHub Action

![Tests](https://github.com/canonical/setup-maas/actions/workflows/test.yml/badge.svg)

A GitHub Action for installing and configuring MAAS

## Usage

### Setup

#### Commands

##### MAAS with empty test database

```yaml
- name: Setup MAAS
  uses: canonical/setup-maas@main
```

##### MAAS with pre-populated test database

```yaml
- name: Setup MAAS
  uses: canonical/setup-maas@main
  with:
    use-maasdb-dump: true
```

#### Create a new user

To add a new user, use the `maas createadmin` command in the next step

```yaml
- name: Create MAAS admin
  run: sudo maas createadmin --username=admin --password=test --email=fake@example.org
```
