```
ssh-keygen -t ed25519 -C "ваша-почта@example.com"
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519
cat ~/.ssh/id_ed25519.pub

ssh -T git@github.com
```
