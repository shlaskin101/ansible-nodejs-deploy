# Automate Nexus Deployment (Ansible)

Automate provisioning and deployment of a Node.js app to a remote Linux VM (DigitalOcean) using **Ansible**. This repo turns the manual install steps into a repeatable playbook you can run on any server.

> **Tech stack:** Ansible • Node.js • Linux (Ubuntu) • DigitalOcean

---

## What this project does

- Installs system dependencies (Node.js & npm)
- Creates a dedicated Linux user for the app
- Uploads/unwraps a packaged Node.js app tarball
- Installs app dependencies with `npm`
- Starts the app and verifies it’s running

> **Lesson takeaway (Nana’s theme):** anything you install/configure manually can be translated into an idempotent Ansible playbook. With infra as code, you can rebuild the same environment in minutes if a server fails.

---

## Repo layout (suggested)

```
ansible/
├── hosts                    # Inventory (your server IP/hostnames)
├── deploy-node.yaml         # Main playbook
├── project-vars.yml         # Variables used by the playbook
└── nodejs-app-1.0.0.tgz     # Example packaged app artifact (optional, or fetch from CI)
```

---

## Prerequisites

- A reachable Linux VM (e.g., DigitalOcean droplet) with SSH access
- Your SSH key configured for the remote user (e.g., `root`)
- Ansible installed locally (`ansible --version`)
- A packaged Node.js app tarball (e.g., `nodejs-app-1.0.0.tgz`) that contains at minimum:

  - `package/` → `app/` → `server` (your app’s start script)
  - `package/` → `package.json`

---

## Inventory (`ansible/hosts`)

```ini
[node_app]
157.230.217.74 ansible_user=root ansible_ssh_private_key_file=~/.ssh/id_ed25519
```

Replace the IP, user, and key path with your values.

---

## Variables (`ansible/project-vars.yml`)

```yaml
version: "1.0.0"
# Where the tarball lives on your local controller (adjust to your path)
location: "/Users/your-username/ansible/nodejs-app"
linux_name: "stevenlaskin" # the app user to create on the server
user_home_dir: "/home/stevenlaskin" # home dir for that user
```

> Keep variables simple YAML. Avoid templating (no `{{ }}`) **inside** the vars file.

---

## Playbook (`ansible/deploy-node.yaml`)

```yaml
---
- name: Install node and npm
  hosts: node_app
  tasks:
    - name: Update apt repo and cache
      apt: update_cache=yes force_apt_get=yes cache_valid_time=3600

    - name: Install node.js and npm
      apt:
        pkg:
          - nodejs
          - npm

- name: Create new linux user for node app
  hosts: node_app
  vars_files:
    - project-vars.yml
  tasks:
    - name: Create linux user
      user:
        name: "{{ linux_name }}"
        comment: Node User
        group: admin

- name: Deploy nodejs app
  hosts: node_app
  become: true
  become_user: "{{ linux_name }}"
  vars_files:
    - project-vars.yml
  tasks:
    - name: Unpack the nodejs file
      unarchive:
        src: "{{ location }}/nodejs-app-{{ version }}.tgz"
        dest: "{{ user_home_dir }}"

    - name: Install dependencies
      npm:
        path: "{{ user_home_dir }}/package"

    - name: Start the application (background)
      command:
        chdir: "{{ user_home_dir }}/package/app"
        cmd: node server
      async: 1000
      poll: 0

    - name: Ensure app is running
      shell: ps aux | grep node
      register: app_status

    - name: Show process output
      debug:
        msg: "{{ app_status.stdout_lines }}"
```

> **Why `become_user`:** we run the app as the dedicated user for least-privilege and cleaner file ownership.

---

## How to run

From your local machine:

```bash
cd ansible
ansible-playbook -i hosts deploy-node.yaml
```

### Example successful output (truncated)

```
PLAY [Deploy nodejs app] ***************************************************************************************************
TASK [Ensure app is running] ***********************************************************************************************
changed: [157.230.217.74]
TASK [Show process output] *************************************************************************************************
ok: [157.230.217.74] => {
  "msg": [
    "stevenl+  107587 25.2  6.7 1055400 66272 ? Sl 14:53 0:00 node server",
    "stevenl+  107619  0.0  0.1   2888  1840 pts/2 S+ 14:53 0:00 /bin/sh -c ps aux | grep node",
    "stevenl+  107621  0.0  0.2   7300  2180 pts/2 S+ 14:53 0:00 grep node"
  ]
}
```

### Quick verification

SSH into the server and run:

```bash
ps aux | grep "node server"
# or
curl -I http://127.0.0.1:YOUR_APP_PORT/
```

---

## Common pitfalls & fixes

- **Undefined variable (e.g., `linux_name is undefined`)**

  - Ensure `vars_files:` points to a real YAML file:

    ```yaml
    vars_files:
      - project-vars.yml
    ```

  - File must live where Ansible can find it (same dir as playbook is simplest).

- **Unarchive can’t find the tarball**

  - Error mentions controller path — confirm `location` points to a real path **on your local machine**, not the remote server.
  - If the tarball already lives **on the remote**, add `remote_src: yes` and update `src` accordingly.

- **App won’t start / permission denied**

  - Verify the app directory is owned by the app user and you’re using `become_user: {{ linux_name }}` for runtime tasks.

- **Need to re-run only the app start & verify**

  - Use tags (optional improvement) to run just a subset of tasks.

---

## How this maps to manual steps

1. Update package cache & install Node.js/npm → **Ansible `apt`** tasks
2. Create a dedicated Linux user → **Ansible `user`**
3. Copy & extract app artifact → **Ansible `unarchive`**
4. Install app deps → **Ansible `npm`**
5. Start process in background & verify → **Ansible `command`**, `shell`, `debug`

**Why it’s better:** idempotent, repeatable, and version-controlled. If the droplet breaks, you redeploy with one command.

---

## Next steps / nice-to-haves

- Use **systemd** to manage the Node app as a service
- Add **handlers** to restart only when code changes
- Parameterize environment (dev/stage/prod) via group_vars
- Wire into CI to build the tarball and run the playbook after artifacts publish

---

## License

MIT (or your choice)
