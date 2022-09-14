## Host Environment Setup

1. Checkout this branch

2. Configure Kerberos credential cache name in /etc/krb5.conf

example:
[libdefaults]
 default_ccache_name = FILE:/tmp/krb5cc_wclark

3. Patch Vagrant to fix https://github.com/hashicorp/vagrant/issues/12884

sed -i.bak 's/elsif info[:keys_only]/else/g' $(sudo rpm -ql vagrant | grep 'lib/vagrant/machine.rb')

4. Inspect vagrant/boxes.d/98-hotfix-build-env.yaml

4.a. replace DEV_USER with your kerberos name
4.b. replace DEV_GITHUB_USER with your github username

## Provision Hotfix Build Environment

1. On your workstation, kinit to obtain kerberos TGT

2. vagrant up fedora36-hotfix-build

3. when it completes, vagrant ssh fedora36-hotfix-build

## Build the Hotfix (foreman with https://bugzilla.redhat.com/2122175 on Satellite 6.11.2)

### 1. Find the foreman version we are building at http://ohsnap.sat.engineering.redhat.com/streams/6.11/releases/6.11.2/snaps/2.0/all_packages

### 2. Create a hotfix branch in satellite-packaging repository

cd satellite-packaging/
git checkout -b hotfix_2122175-sat-6.11.2 SATELLITE-6.11.0
git reset --hard $(git log --grep '3.1.1.22' --oneline | awk '{Print $1}' | head -n1)
vi packages/foreman/foreman.spec # bump the release 1 -> 2.HOTFIXRHBZ2122175
^vi^git add
git rm packages/foreman/foreman-3.1.1.22*
git commit -m "Hotfix BZ 2122175 Satellite 6.11.2"

### 3. Create hotfix source branch in foreman repository

cd ../foreman
HOTFIX_COMMIT=$(git log --grep 35346 --oneline | awk '{Print $1}' | head -n1)
git checkout -b hotfix_2122175-sat-6.11.2 SATELLITE-6.11.0
git reset --hard $(git log --grep '3.1.1.22' --oneline | awk '{Print $1}' | head -n1)
git cherry-pick -x $HOTFIX_COMMIT
git tag -f 3.1.1.22 HEAD

### 4. Build Foreman source

cd ../tool_belt/
bundle exec bin/tool-belt release build-source --type tar --dir ../foreman
cd ../foreman
git tag -f 3.1.1.22 HEAD~1

### 5. Add new Foreman source to hotfix branch in packaging

cd ../satellite-packaging
cp ../foreman/foreman-3.1.1.22* packages/foreman/
git add packages/<tabtabtabtab>
git commit --amend --no-edit

### 6. Build Hotfix RPM

source env/bin/activate
obal scratch foreman
