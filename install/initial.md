In order to install a new Red Hat Openshift cluster in AWS please follow these steps in the precise order.

You will need the Pull Secret generated in this page:
* https://cloud.redhat.com/openshift/install

All the steps will be performed from an AWS Cloud9 terminal with enough privileges (AdministratorAccess will work):
```bash
file=policy.yaml
wget https://raw.githubusercontent.com/secobau/openshift/master/etc/aws/$file
aws cloudformation create-stack --stack-name ocp-${file%.yaml} --template-body file://$file --capabilities CAPABILITY_NAMED_IAM


```
You will need to obtain a valid public domain name before installing the cluster:
* https://console.aws.amazon.com/route53/home

Disable AWS managed temporary credentials in AWS Cloud9 settings. Now you need to create a new Access Key in your AWS IAM Security Credentials and then configure your AWS Cloud9 terminal:
```bash
aws configure


```

Now generate an SSH key pair and add it to the SSH agent. This will allow you to access the cluster nodes through SSH.
```bash
ssh-keygen

eval "$(ssh-agent -s)"
ssh-add $HOME/.ssh/id_rsa


```
Choose a version number:
```bash
export version=4.5.13


```
Afterwards you can proceed to install the client and the installer binaries:
```bash
for mode in client install
do
  wget https://mirror.openshift.com/pub/openshift-v4/clients/ocp/$version/openshift-$mode-linux-$version.tar.gz
  gunzip openshift-$mode-linux-$version.tar.gz
  tar xf openshift-$mode-linux-$version.tar
  rm openshift-$mode-linux-$version.tar
done
mkdir --parents $HOME/bin
for binary in kubectl oc
do
  mv $binary $HOME/bin
done
mv openshift-install $HOME/bin/openshift-install-$version
file=README.md 
test -f $file && rm -f $file
file=$HOME/bin/openshift-install
test -f $file && rm -f $file
ln -s $HOME/bin/openshift-install-$version $HOME/bin/openshift-install
rm $HOME/bin/kubectl && ln -s $HOME/bin/oc $HOME/bin/kubectl    


```
Now you introduce your choice for the name and domain of the cluster:
```bash
export ClusterName=openshift
export DomainName=sebastian-colomar.es


```
Create a directory to place all the configuration files:
```bash
export dir="$HOME/environment/openshift/install/$ClusterName.$DomainName"
test -d $dir || mkdir --parents $dir


```
Now you create a configuration file template to be later modified:
```bash
openshift-install-$version create install-config --dir $dir --log-level debug


```
It is optionally a good idea to initialize a git repository to track history of the configuration files:
```bash
cd $dir && git init
git config --global user.name "Your Name"
git config --global user.email you@example.com
git add .
git commit -m Initial


```
The following script will modify the EC2 instance type so as to choose the cheapest possible type but big enough to correctly set up the cluster:
```bash
cd $dir
wget https://raw.githubusercontent.com/secobau/openshift/master/install/fix-config.sh
chmod +x fix-config.sh && ./fix-config.sh && rm fix-config.sh
git commit -am 'Set EC2 instance type'


```
If you wish your cluster to be private and not accessible from the external network:
```bash
export Publish=Internal
sed --in-place s/External/$Publish/ $dir/install-config.yaml
git commit -am 'Set Publish value'


```
Otherwise set the publish option to be external:
```bash
export Publish=External


```
