# The malicious dojo — push this to GitHub

Two files, pushed in two phases. **Order matters**: if `files:` is present when you
create the dojo, `dojo_create` runs `dojo_initialize_files` under *your* session,
hits `assert is_admin()` at `utils/dojo.py:309`, and the whole create fails with

```
yml-specified files support requires admin privileges
```

That assert is the real gate on this bug, and it is working as intended. The attack
does not defeat it — it borrows an **admin's** session instead.

| File | Role |
|---|---|
| `dojo.yml` | phase 1 — no `files:` block, so an ordinary user can create the dojo |
| `dojo.phase2.yml` | phase 2 — same dojo plus the `files:` payload; push this *after* the dojo exists |
| `DESCRIPTION.md` | placeholder; `arm.py` writes the real one into the database |
| `module/challenge/` | makes the dojo structurally valid |

## Sequence

**1 — push phase 1 and create the dojo** (ordinary account, no admin rights)

```bash
cp dojo.yml /path/to/your/repo/dojo.yml   # already the phase-1 version
git -C /path/to/your/repo add -A && git -C /path/to/your/repo commit -m "my dojo" && git push
```

Create it at `/dojos/create` with your repo, e.g. `alice/totally-normal-dojo`.
Note the reference id it returns, e.g. `totally-normal-dojo~a1b2c3d4`.

**2 — arm it**

```bash
python3 ../arm.py --dojo totally-normal-dojo~a1b2c3d4 --user alice
```

This reads your own `update_code` from `/dojo/<ref>/admin/` — `dojo_admin.html:25`
renders that link for the dojo's owner, with no `is_admin()` guard — and then sets
the dojo description to a 1×1 `<img>` pointing at your own update URL, via
`POST /pwncollege_api/v1/dojos/<dojo>/update`, which is gated by `dojo_admins_only`,
meaning you. Both steps verified working from a non-admin account.

**3 — push phase 2**

```bash
cp dojo.phase2.yml /path/to/your/repo/dojo.yml
git -C /path/to/your/repo commit -am "tweak" && git push
```

Do **not** open the update URL yourself. Your session is not an admin, so it returns
400 and writes nothing.

**4 — wait**

An admin opens `/<your-ref>/`. Their browser fetches the `<img>` as an ordinary
same-origin subresource — no CSRF token, no form, no click — `update_dojo` re-pulls
your repo under their session, `is_admin()` is now true, and the payload is written
as root.

```bash
docker exec dojo docker exec ctfd cat /tmp/POC_D01_PWNED
# pwned as uid=0 via ['-c']
```

`/admin/dojos` lists every dojo as a link to its public page, so that click is one
step from the admin index.

## Payload

`dojo.phase2.yml` writes an inert marker via a `.pth` file in the CTFd venv. It
executes on every Python start in that container — including the Docker healthcheck,
which runs every 10 seconds — until removed. Clean up with
`python3 ../host_dojo.py --reset` or `python3 ../../cleanup.py`.
