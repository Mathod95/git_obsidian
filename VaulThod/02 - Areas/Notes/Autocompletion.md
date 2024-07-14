Kubectl est fourni avec de l'autocomplétion, c'est pratique et ça permet d'aller très vite dans la création des commandes kubectl :

```shell ln:false
sudo apt install bash-completion
kubectl completion bash | sudo tee /etc/bash_completion.d/kubectl > /dev/null
```
Redémarrer votre shell et tapez `kubectl` suivi de tabulation. Vous ferez par exemple pour changer de namespace, on verra la création plus tard, il suffit de taper kubectl -n tab et la liste de tous les namespaces existant sur votre cluster s'affichera.
```shell ln:false
kubectl -n
default          kube-node-lease  kube-public      kube-system
kubectl config set-context --current --namespace=kube-system
Context "kubernetes-admin@kubernetes" modified.
```
Pour aller encore plus on peut utiliser un alias :
```shell ln:false
alias k=kubectl
complete -F __start_kubectl k
k tab
annotate       apply          autoscale      completion
```
Pour le rendre permanent ajouter ces deux lignes à la fin de votre fichier `.bashrc`