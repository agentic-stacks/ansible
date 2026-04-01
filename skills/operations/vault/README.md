# Vault

## Encrypt a File

Encrypt an entire file with Ansible Vault. Ansible replaces the plaintext content with an encrypted blob that is safe to commit.

```bash
ansible-vault encrypt secrets.yml
```

Ansible prompts for a vault password, then rewrites the file in place. The file header becomes `$ANSIBLE_VAULT;1.1;AES256`.

```bash
# Encrypt multiple files at once
ansible-vault encrypt group_vars/prod/secrets.yml group_vars/staging/secrets.yml
```

> **Warning** Never commit unencrypted secrets. If you see plaintext secrets, encrypt them immediately.

## Decrypt a File

Decrypt a vault-encrypted file back to plaintext.

```bash
# Decrypt in place — overwrites the file with plaintext
ansible-vault decrypt secrets.yml

# View without decrypting the file on disk
ansible-vault view secrets.yml
```

`ansible-vault view` prints the decrypted content to stdout. The file on disk stays encrypted. Use `view` when you need to inspect secrets without risk of accidentally committing plaintext.

## Edit an Encrypted File

Open an encrypted file in your editor, make changes, and re-encrypt on save.

```bash
ansible-vault edit secrets.yml
```

Ansible decrypts the file to a temporary location, opens it in `$EDITOR` (falls back to `vi`), and re-encrypts when the editor exits. The plaintext never touches disk in the working directory.

## Encrypt a Single Variable

Encrypt an individual value instead of an entire file. This lets you mix encrypted and unencrypted variables in the same file.

```bash
ansible-vault encrypt_string 'secret_value' --name 'my_secret'
```

Output:

```yaml
my_secret: !vault |
          $ANSIBLE_VAULT;1.1;AES256
          62313365396662343061393464336163383764316462376134613633303133376331376333613361
          6234373734376536643234376538363965613832346432330a623832353831613032636338633763
          37643966326437643233643539303039313464373661363664353838653932363330373539616232
          3935623062353965300a343164393230303765646433613037333638313665373063343263623966
          3335
```

Paste the output directly into a variables file. Ansible decrypts it at runtime.

```bash
# Encrypt with a vault ID
ansible-vault encrypt_string --vault-id dev@prompt 'secret_value' --name 'my_secret'

# Read the value from stdin
echo -n 'secret_value' | ansible-vault encrypt_string --stdin-name 'my_secret'
```

## Use Vault in Playbooks

Pass the vault password when running a playbook so Ansible can decrypt secrets at runtime.

```bash
# Prompt for the vault password interactively
ansible-playbook site.yml --ask-vault-pass

# Read the password from a file
ansible-playbook site.yml --vault-password-file .vault_pass

# Set the password file via environment variable
export ANSIBLE_VAULT_PASSWORD_FILE=.vault_pass
ansible-playbook site.yml
```

The `ANSIBLE_VAULT_PASSWORD_FILE` environment variable eliminates the need to pass `--vault-password-file` on every command. Set it in your shell profile or CI environment.

## Use Vault IDs (Multi-Password)

Vault IDs let you use different passwords for different environments or sensitivity levels.

```bash
# Encrypt with a named vault ID, prompting for the password
ansible-vault encrypt --vault-id dev@prompt secrets_dev.yml

# Encrypt with a named vault ID, reading the password from a file
ansible-vault encrypt --vault-id prod@/path/to/prod-vault-pass secrets_prod.yml
```

Run a playbook that contains secrets encrypted with multiple vault IDs:

```bash
ansible-playbook site.yml \
  --vault-id dev@.vault_pass_dev \
  --vault-id prod@.vault_pass_prod
```

**When to use vault IDs:** Separate vault passwords for dev, staging, and prod environments. This limits exposure when a password is compromised — only secrets encrypted with that vault ID are at risk. It also lets different teams hold different passwords.

## Create a Vault Password File

A vault password file is a plain text file containing the password on a single line, or an executable script that outputs the password.

### Plain text file

```bash
echo 'my-vault-password' > .vault_pass
chmod 600 .vault_pass
```

### Executable script

```bash
#!/bin/bash
# .vault_pass.sh — retrieve the vault password from a secrets manager
# Must be executable: chmod 755 .vault_pass.sh

# Example: read from an environment variable
echo "$VAULT_PASSWORD"

# Example: read from a secrets manager
# aws secretsmanager get-secret-value --secret-id ansible-vault --query SecretString --output text
```

```bash
chmod 755 .vault_pass.sh
ansible-playbook site.yml --vault-password-file .vault_pass.sh
```

Ansible detects that the file is executable and runs it to obtain the password. The script must output the password on a single line to stdout.

> **Warning** Set restrictive permissions on vault password files. Use `chmod 600` for plain text files and `chmod 755` for scripts. Never commit vault password files to version control.

## Integrate Vault in CI/CD

Pass the vault password through an environment variable in CI/CD pipelines. Write it to a temporary file, run the playbook, then clean up.

```bash
# Write the vault password from an environment variable to a file
echo "$VAULT_PASS" > .vault_pass
chmod 600 .vault_pass

# Run the playbook
ansible-playbook site.yml --vault-password-file .vault_pass

# Clean up the password file
rm -f .vault_pass
```

Add the password file to `.gitignore` to prevent accidental commits:

```
# .gitignore
.vault_pass
.vault_pass.*
```

For multi-environment pipelines, use vault IDs:

```bash
echo "$VAULT_PASS_DEV" > .vault_pass_dev
echo "$VAULT_PASS_PROD" > .vault_pass_prod
chmod 600 .vault_pass_dev .vault_pass_prod

ansible-playbook site.yml \
  --vault-id dev@.vault_pass_dev \
  --vault-id prod@.vault_pass_prod

rm -f .vault_pass_dev .vault_pass_prod
```

## Rekey Encrypted Files

Change the password on vault-encrypted files with `rekey`. Ansible decrypts with the old password and re-encrypts with a new one.

```bash
ansible-vault rekey secrets.yml

# Rekey multiple files
ansible-vault rekey group_vars/prod/secrets.yml group_vars/staging/secrets.yml

# Rekey with vault IDs
ansible-vault rekey --vault-id prod@prompt secrets_prod.yml
```

**When to rekey:**

- A team member with vault access leaves the organization
- A vault password is compromised or suspected of compromise
- Password rotation policy requires periodic changes
- Migrating from a single vault password to vault IDs

After rekeying, update all vault password files and CI/CD secrets to use the new password.

## References

- [Protecting sensitive data with Ansible Vault](https://docs.ansible.com/ansible/latest/vault_guide/index.html)
- [Encrypting content with Ansible Vault](https://docs.ansible.com/ansible/latest/vault_guide/vault_encrypting_content.html)
- [Managing vault passwords](https://docs.ansible.com/ansible/latest/vault_guide/vault_managing_passwords.html)
- [Using encrypted variables and files](https://docs.ansible.com/ansible/latest/vault_guide/vault_using_encrypted_content.html)
- [ansible-vault CLI](https://docs.ansible.com/ansible/latest/cli/ansible-vault.html)
