# Operator learning

## Rely On
```shell
1. git
2. go 1.19+ ( 需要设置 GOPROXY )
3. kubectl
```

## Install operator-sdk
```shell
mkdir -p $GOPATH/src/github.com/operator-framework
cd $GOPATH/src/github.com/operator-framework
git clone https://github.com/operator-framework/operator-sdk
cd operator-sdk
git checkout master
make install
```

## Use Operator-SDK Create Mysql-operator
```shell
mkdir -p $GOPATH/src/github.com/example-inc/
cd $GOPATH/src/github.com/example-inc/
operator-sdk init --domain example.com --repo github.com/chengxiaobai/mysql-operator
operator-sdk create api --group mysql --version v1beta1 --kind mysqlcluster
```
