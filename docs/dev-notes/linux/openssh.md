# OpenSSH

## Cheatsheet

- generate ssh key
  
  ```bash
  mkdir ~/.ssh
  ssh-keygen -t ed25519 -C "[user@server]" -f ~/.ssh/[server]/id_ed25519
  ```

- create the `authorized_keys`
  
  ```bash
  touch ~/.ssh/authorized_keys
  echo "[public-key-sting]" >> ~/.ssh/authorized_keys
  type ~/.ssh/[server]/id_ed25519.pub | ssh [user]@[server] "cat >> ~/.ssh/authorized_keys"
  ```

- set permissions/ownership on `.ssh`
  
  ```bash
  chmod 700 -R ~/.ssh
  chmod 600 ~/.ssh/config
  chmod 644 ~/.ssh/*.pub
  chmod 644 ~/.ssh/authorized_keys
  chmod 644 ~/.ssh/known_hosts
  chown [user]:[group] ~/.ssh/authorized_keys
  ll -R ~/.ssh | grep 'ssh\|auth'
  ```

- verify permissions/ownership on `.ssh`
  
  ```bash
  ll -R ~ | grep 'ssh\|auth'
  ```

## SSH Hardening

- disable password login in `sshd_config`
  
  ```bash
  sudo vi /etc/ssh/sshd_config
  # uncomment and change: 
    '#PasswordAuthentication yes' -> 'PasswordAuthentication no'
  sudo systemctl restart ssh
  ```
  
   > 
   > \[!danger\]
   > Open new SSH season and test login with RSA Keys before **closing** the existing connection

- change default ssh port in `sshd_config`
  
  ```bash
  sudo vi /etc/ssh/sshd_config
  # change line
    'port 1337'
  sudo systemctl restart ssh
  ```

## References

- [3os Project](https://3os.org/): technical documentation/guides for DevOps engineers/sysadmins
