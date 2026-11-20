# Commands i used for this lab latter will be explained
## ssh key generation for the lab but inside my mac.

### ssh key generation
```
    ssh-keygen -t ed25519 -f ~/.ssh/lab_key -C "lab-server"
```
### ssh key copying
```
    cat ~/.ssh/lab_key.pub
```


### So you used the right key, but by luck. On a server that allowed only a few failed attempts, SSH trying several keys could also lock you out. That's why the clean command is better:
```
    ssh -i ~/.ssh/lab_key -o IdentitiesOnly=yes root@LAB_IP
```