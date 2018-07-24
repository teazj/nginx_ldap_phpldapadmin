## 初始化ldap,创建schema

docker exec -ti openldap-server bash
mkdir ~/xiaoniu_tokens
cat <<EOF > ~/xiaoniu_tokens/xiaoniuToken.schema
attributeType ( 1.3.6.1.4.1.18171.2.1.8
        NAME 'xiaoniuToken'
        DESC 'xiaoniu authentication token'
        EQUALITY caseExactIA5Match
        SUBSTR caseExactIA5SubstringsMatch
        SYNTAX 1.3.6.1.4.1.1466.115.121.1.26 SINGLE-VALUE )

objectClass ( 1.3.6.1.4.1.18171.2.3
        NAME 'xiaoniuAuthenticationObject'
        DESC 'Object that may authenticate to a xiaoniu cluster'
        AUXILIARY
        MUST xiaoniuToken )
EOF


echo "include /root/xiaoniu_tokens/xiaoniuToken.schema" > ~/xiaoniu_tokens/schema_convert.conf
slaptest -f ~/xiaoniu_tokens/schema_convert.conf -F ~/xiaoniu_tokens
>~/xiaoniu_tokens/cn=config/cn=schema/cn\=\{0\}xiaoniutoken.ldif
cat <<EOF >  ~/xiaoniu_tokens/cn=config/cn=schema/cn\=\{0\}xiaoniutoken.ldif
# AUTO-GENERATED FILE - DO NOT EDIT!! Use ldapmodify.
# CRC32 e0d7fb85
dn: cn=xiaoniutoken,cn=schema,cn=config
objectClass: olcSchemaConfig
cn: xiaoniutoken
olcAttributeTypes: {0}( 1.3.6.1.4.1.18171.2.1.8 NAME 'xiaoniuToken' DESC
 'xiaoniu authentication token' EQUALITY caseExactIA5Match SUBSTR caseExa
 ctIA5SubstringsMatch SYNTAX 1.3.6.1.4.1.1466.115.121.1.26 SINGLE-VALUE )
olcObjectClasses: {0}( 1.3.6.1.4.1.18171.2.3 NAME 'xiaoniuAuthenticationO
 bject' DESC 'Object that may authenticate to a xiaoniu cluster' AUXILIAR
 Y MUST xiaoniuToken )
structuralObjectClass: olcSchemaConfig
entryUUID: 3d87334a-1fc0-1038-9ce2-1f833165819f
creatorsName: cn=config
createTimestamp: 20180719165509Z
entryCSN: 20180719165509.724257Z#000000#000#000000
modifiersName: cn=config
modifyTimestamp: 20180719165509Z
EOF


cd ~/xiaoniu_tokens/cn=config/cn=schema
ldapadd -c -Y EXTERNAL -H ldapi:/// -f cn\=\{0\}xiaoniutoken.ldif

ldapsearch -x -H ldap:/// -LLL -D "cn=admin,cn=config" -w password -b "cn=schema,cn=config" "(objectClass=olcSchemaConfig)" dn -Z



cat <<EOF > groups.ldif
dn: ou=People,dc=xiaoniu,dc=com
ou: People
objectClass: top
objectClass: organizationalUnit
description: Parent object of all UNIX accounts

dn: ou=Groups,dc=xiaoniu,dc=com
ou: Groups
objectClass: top
objectClass: organizationalUnit
description: Parent object of all UNIX groups

dn: cn=ad,ou=Groups,dc=xiaoniu,dc=com
cn: ad
gidnumber: 100
memberuid: user1
memberuid: user2
objectclass: posixGroup
objectclass: top
EOF

ldapmodify -x -a -H ldap:// -D "cn=admin,dc=xiaoniu,dc=com" -w password -f groups.ldif




cat <<EOF > users.ldif
dn: uid=user1,ou=People,dc=xiaoniu,dc=com
cn: user1
gidnumber: 100
givenname: user1
homedirectory: /home/users/user1
loginshell: /bin/sh
objectclass: inetOrgPerson
objectclass: posixAccount
objectclass: top
objectClass: shadowAccount
objectClass: organizationalPerson
sn: user1
uid: user1
uidnumber: 1000
userpassword: user1

dn: uid=user2,ou=People,dc=xiaoniu,dc=com
homedirectory: /home/users/user2
loginshell: /bin/sh
objectclass: inetOrgPerson
objectclass: posixAccount
objectclass: top
objectClass: shadowAccount
objectClass: organizationalPerson
cn: user2
givenname: user2
sn: user2
uid: user2
uidnumber: 1001
gidnumber: 100
userpassword: user2
EOF


ldapmodify -x -a -H ldap:// -D "cn=admin,dc=xiaoniu,dc=com" -w password -f users.ldif


cat <<EOF > users.txt
dn: uid=user1,ou=People,dc=xiaoniu,dc=com
dn: uid=user2,ou=People,dc=xiaoniu,dc=com
EOF


## 执行脚本

while read -r user; do
fname=$(echo $user | grep -E -o "uid=[a-z0-9]+" | cut -d"=" -f2)
token=$(dd if=/dev/urandom bs=128 count=1 2>/dev/null | base64 | tr -d "=+/" | dd bs=32 count=1 2>/dev/null)
cat << EOF > "${fname}.ldif"
$user
changetype: modify
add: objectClass
objectclass: kubernetesAuthenticationObject
-
add: kubernetesToken
kubernetesToken: $token
EOF

ldapmodify -a -H ldapi:/// -D "cn=admin,dc=xiaoniu,dc=com" -w password  -f "${fname}.ldif"
done < users.txt
