# Templates

## Template Module

Use `ansible.builtin.template` to render Jinja2 templates and deploy them to managed hosts.

```yaml
---
- name: Deploy configuration templates
  hosts: webservers
  become: true

  tasks:
    - name: Deploy nginx configuration
      ansible.builtin.template:
        src: nginx.conf.j2
        dest: /etc/nginx/nginx.conf
        owner: root
        group: root
        mode: "0644"
        backup: true
        validate: "nginx -t -c %s"
      notify: Reload nginx

    - name: Deploy application config
      ansible.builtin.template:
        src: app.conf.j2
        dest: /etc/app/config.yml
        owner: app
        group: app
        mode: "0600"
```

| Parameter | Description |
|-----------|-------------|
| `src` | Path to the Jinja2 template file (relative to `templates/` directory or absolute) |
| `dest` | Absolute path on the remote host |
| `owner` | File owner on the remote host |
| `group` | File group on the remote host |
| `mode` | File permissions (quote octal strings like `"0644"`) |
| `backup` | Create a backup of the original file before replacing |
| `validate` | Command to validate the file before moving it into place (`%s` = temp file path) |
| `force` | Replace the file even if it already exists (default: `true`) |
| `newline_sequence` | Newline character sequence (`\n`, `\r`, `\r\n`) |

## Filters

Filters transform variable values within Jinja2 expressions.

### Default and Mandatory

```yaml
# Provide a default value when variable is undefined
- name: Use default filter
  ansible.builtin.debug:
    msg: "Port is {{ http_port | default(80) }}"

# Default only when undefined (not when empty or false)
- name: Default with boolean=true
  ansible.builtin.debug:
    msg: "Debug is {{ debug_mode | default(false, true) }}"

# Fail if variable is not defined
- name: Require a variable
  ansible.builtin.debug:
    msg: "Version is {{ app_version | mandatory }}"
```

### String Filters

```yaml
- name: String manipulation
  ansible.builtin.debug:
    msg: |
      uppercase: {{ name | upper }}
      lowercase: {{ name | lower }}
      replace: {{ path | regex_replace('\.conf$', '.conf.bak') }}
      regex search: {{ log_line | regex_search('ERROR (\d+)', '\1') }}
      capitalize: {{ name | capitalize }}
      trim: {{ user_input | trim }}
```

### Data Format Filters

```yaml
- name: Convert data formats
  ansible.builtin.debug:
    msg: |
      JSON: {{ my_dict | to_json }}
      Pretty JSON: {{ my_dict | to_nice_json }}
      YAML: {{ my_dict | to_yaml }}
      Pretty YAML: {{ my_dict | to_nice_yaml }}

- name: Parse data
  ansible.builtin.set_fact:
    parsed_data: "{{ json_string | from_json }}"
    parsed_yaml: "{{ yaml_string | from_yaml }}"
```

### Collection Filters

```yaml
- name: Combine dictionaries
  ansible.builtin.set_fact:
    merged_config: "{{ defaults | combine(overrides, recursive=True) }}"
  vars:
    defaults:
      port: 80
      logging:
        level: info
    overrides:
      port: 8080
      logging:
        level: debug

- name: List operations
  ansible.builtin.debug:
    msg: |
      mapped: {{ users | map(attribute='name') | list }}
      selected: {{ numbers | select('greaterthan', 5) | list }}
      rejected: {{ items | reject('match', '^test') | list }}
      unique: {{ all_items | unique | list }}
      flatten: {{ nested_list | flatten }}
      sorted: {{ names | sort }}
      first: {{ items | first }}
      last: {{ items | last }}
      length: {{ items | length }}
```

### Ternary and Type Filters

```yaml
- name: Ternary operator
  ansible.builtin.debug:
    msg: "{{ (env == 'production') | ternary('https', 'http') }}"

- name: Type casting
  ansible.builtin.debug:
    msg: |
      integer: {{ string_num | int }}
      float: {{ string_num | float }}
      boolean: {{ value | bool }}
      string: {{ number | string }}

- name: Math filters
  ansible.builtin.debug:
    msg: |
      abs: {{ value | abs }}
      round: {{ float_val | round(2) }}
      max of list: {{ [3, 1, 4, 1, 5] | max }}
      min of list: {{ [3, 1, 4, 1, 5] | min }}
```

### Path and Hash Filters

```yaml
- name: Path manipulation
  ansible.builtin.debug:
    msg: |
      basename: {{ "/etc/nginx/nginx.conf" | basename }}
      dirname: {{ "/etc/nginx/nginx.conf" | dirname }}
      expanduser: {{ "~/.ssh/id_rsa" | expanduser }}
      realpath: {{ symlink_path | realpath }}

- name: Hash and encoding
  ansible.builtin.debug:
    msg: |
      sha256: {{ "secret" | hash('sha256') }}
      md5: {{ "data" | hash('md5') }}
      base64: {{ "plaintext" | b64encode }}
      decode: {{ encoded_string | b64decode }}
      password: {{ "mypassword" | password_hash('sha512', 'salt') }}
```

## Tests

Tests check values in conditional expressions.

```yaml
- name: Check if defined
  ansible.builtin.debug:
    msg: "Variable exists"
  when: my_var is defined

- name: Check if undefined
  ansible.builtin.debug:
    msg: "Variable does not exist"
  when: optional_var is undefined

- name: Truthy and falsy
  ansible.builtin.debug:
    msg: "Value is truthy"
  when: my_var is truthy

- name: Check falsy
  ansible.builtin.debug:
    msg: "Value is falsy"
  when: my_var is falsy

- name: Pattern matching
  ansible.builtin.debug:
    msg: "Matches pattern"
  when: hostname is match("^web\d+")

- name: Search for substring
  ansible.builtin.debug:
    msg: "Contains error"
  when: log_output is search("ERROR")

- name: Version comparison
  ansible.builtin.debug:
    msg: "Version is new enough"
  when: ansible_facts['distribution_version'] is version('22.04', '>=')

- name: Subset and superset
  ansible.builtin.set_fact:
    required_packages:
      - nginx
      - python3
    installed_packages:
      - nginx
      - python3
      - git
      - curl

- name: Check if required packages are all installed
  ansible.builtin.debug:
    msg: "All required packages are installed"
  when: required_packages is subset(installed_packages)

- name: Check superset
  ansible.builtin.debug:
    msg: "Installed packages contain all required ones"
  when: installed_packages is superset(required_packages)

- name: Check file path
  ansible.builtin.debug:
    msg: "Path is absolute"
  when: file_path is abs

- name: String tests
  ansible.builtin.debug:
    msg: "Value is a string"
  when: my_var is string

- name: Number test
  ansible.builtin.debug:
    msg: "Value is a number"
  when: my_var is number
```

## Lookups

Lookups retrieve data from external sources and are executed on the Ansible controller.

### File Lookup

```yaml
- name: Read a local file
  ansible.builtin.debug:
    msg: "{{ lookup('ansible.builtin.file', '/etc/hostname') }}"

- name: Use file content as a variable
  ansible.builtin.copy:
    content: "{{ lookup('ansible.builtin.file', 'files/ssh_key.pub') }}"
    dest: /home/deploy/.ssh/authorized_keys
    mode: "0600"
```

### Environment Lookup

```yaml
- name: Read environment variable
  ansible.builtin.debug:
    msg: "Home directory is {{ lookup('ansible.builtin.env', 'HOME') }}"

- name: Use CI variable
  ansible.builtin.debug:
    msg: "Build number {{ lookup('ansible.builtin.env', 'BUILD_NUMBER') }}"
  when: lookup('ansible.builtin.env', 'CI') | bool
```

### Pipe Lookup

```yaml
- name: Run a command and use its output
  ansible.builtin.debug:
    msg: "Git SHA is {{ lookup('ansible.builtin.pipe', 'git rev-parse HEAD') }}"

- name: Get current timestamp
  ansible.builtin.set_fact:
    deploy_time: "{{ lookup('ansible.builtin.pipe', 'date +%Y%m%d-%H%M%S') }}"
```

### Password Lookup

```yaml
- name: Generate a random password
  ansible.builtin.user:
    name: deploy
    password: "{{ lookup('ansible.builtin.password', '/tmp/deploy_password length=20 chars=ascii_letters,digits') | password_hash('sha512') }}"

- name: Generate password without saving to file
  ansible.builtin.set_fact:
    db_password: "{{ lookup('ansible.builtin.password', '/dev/null length=32 chars=ascii_letters,digits,punctuation') }}"
```

### Template Lookup

```yaml
- name: Render a template string
  ansible.builtin.debug:
    msg: "{{ lookup('ansible.builtin.template', 'message.j2') }}"

- name: Use template lookup in a variable
  ansible.builtin.set_fact:
    rendered_config: "{{ lookup('ansible.builtin.template', 'config.yml.j2') }}"
```

### URL Lookup

```yaml
- name: Fetch content from a URL
  ansible.builtin.debug:
    msg: "{{ lookup('ansible.builtin.url', 'https://api.example.com/version') }}"

- name: Fetch with headers
  ansible.builtin.set_fact:
    api_response: "{{ lookup('ansible.builtin.url', 'https://api.example.com/data', headers={'Authorization': 'Bearer ' + api_token}) }}"
```

## Template Best Practices

### Use ansible_managed

Include `{{ ansible_managed }}` at the top of every template to mark files as generated. This prevents manual edits to managed files.

```jinja
# {{ ansible_managed }}
# Do not edit this file manually — it is managed by Ansible.

server {
    listen {{ http_port }};
    server_name {{ server_name }};

    location / {
        proxy_pass http://127.0.0.1:{{ app_port }};
    }
}
```

### Use the validate Parameter

Validate generated configuration before replacing the live file. If validation fails, Ansible does not deploy the file.

```yaml
- name: Deploy sudoers file with validation
  ansible.builtin.template:
    src: sudoers.j2
    dest: /etc/sudoers
    validate: "visudo -cf %s"

- name: Deploy sshd config with validation
  ansible.builtin.template:
    src: sshd_config.j2
    dest: /etc/ssh/sshd_config
    validate: "sshd -t -f %s"
  notify: Restart sshd

- name: Deploy Apache config with validation
  ansible.builtin.template:
    src: httpd.conf.j2
    dest: /etc/httpd/conf/httpd.conf
    validate: "httpd -t -f %s"
  notify: Reload Apache
```

### Use the backup Parameter

Keep a backup of the original file so you can restore it if needed.

```yaml
- name: Deploy config with backup
  ansible.builtin.template:
    src: app.conf.j2
    dest: /etc/app/config.yml
    backup: true
```

Ansible creates a timestamped backup file in the same directory (for example, `/etc/app/config.yml.2024-01-15@10:30:00~`).

### Template Organization

```
roles/
  nginx/
    templates/
      nginx.conf.j2
      site.conf.j2
    defaults/
      main.yml        # default values for template variables
    tasks/
      main.yml
    handlers/
      main.yml
```

### Control Whitespace

Use Jinja2 whitespace control to produce clean output.

```jinja
# {{ ansible_managed }}

{% for vhost in virtual_hosts %}
server {
    listen {{ vhost.port }};
    server_name {{ vhost.name }};
{%- if vhost.ssl | default(false) %}
    ssl_certificate {{ vhost.ssl_cert }};
    ssl_certificate_key {{ vhost.ssl_key }};
{%- endif %}

{% for location in vhost.locations | default([]) %}
    location {{ location.path }} {
        proxy_pass {{ location.backend }};
    }
{% endfor %}
}
{% endfor %}
```

## References

- [Templating (Jinja2)](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_templating.html)
- [Using Filters to Manipulate Data](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_filters.html)
- [Tests](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_tests.html)
- [Lookups](https://docs.ansible.com/ansible/latest/plugins/lookup.html)
- [ansible.builtin.template module](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/template_module.html)
