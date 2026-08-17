# The malicious POC dojo for RCE in CTFd container on pwn.college

| File | Role |
|---|---|
| `safe.dojo.yml` | phase 1 — no `files:` block, so an ordinary user can create the dojo |
| `poc.dojo.yml` | phase 2 — same dojo plus the payload |
| `DESCRIPTION.md` | holds the `__UPDATE__LINK__` placeholder you fill in at step 2 |


## Exploit

**1 - Create the dojo**

- Fork this dojo
- run
```bash
cp safe.dojo.yml dojo.yml   # start with the safe version
```
- And create a new dojo with a regular account.

**2 - Arm the dojo**

- With the dojo created, goto dojo admin page, copy the update link and replace `__UPDATE__LINK__` in DESCRIPTION.md with it.
- Commit the changes and update the dojo.

**3 - push phase 2**

- Update the dojo.yml to the poc:
```bash
cp poc.dojo.yml dojo.yml
```
- Add and commit the changes.

**4 - wait for admin to open the dojo**

When an admin opens the dojo, their browser fetches the `<img>` as an ordinary
same-origin subresource — no form, no click, no CSRF token. `update_dojo` has no auth
decorator and is CSRF-exempt, so it re-pulls your repo under their session, where
`is_admin()` is now true, and writes the payload as root.

**Verify**

```bash
docker exec dojo docker exec ctfd cat /tmp/POC_D01_PWNED
# pwned as uid=0 via ['-c']
```


## Payload

`poc.dojo.yml` writes an inert marker via a `.pth` file in the CTFd venv. It
executes on every Python start in that container — including the Docker healthcheck,
which runs every 10 seconds — until removed.
