# redis-cli-for-PAS
This is a version of the redis-cli that's compiled to run on a PAS w/ cflinuxfs3 AI, accompanied with instructions about how to deploy and use it, and how to rebuild it if desired

# How to deploy
1. clone this repo into an empty directory and cd into the directory containing the "redis-cli" file
2. cf push redcli -p . -b binary_buildpack --no-start
3. cf set-health-check redcli none
4. cf push redcli -p . -b binary_buildpack -c "while true; do echo 'sleeping'; sleep 2; done"

when the above steps are finished, the app should be running and you should see output like
~~~~
name:                redcli
requested state:     started
isolation segment:   iso-01
routes:              redcli.apps.pcfone.io
last uploaded:       Thu 19 Dec 15:56:04 EST 2019
stack:               cflinuxfs3
buildpacks:          binary

type:            web
instances:       1/1
memory usage:    1024M
start command:   while true; do echo 'sleeping'; sleep 2; done
     state     since                  cpu    memory       disk         details
#0   running   2019-12-19T20:56:13Z   0.2%   8.8M of 1G   6.5M of 1G
~~~~

# How to use
1. cf bind-service redcli *redis-service-name*
2. cf restart redcli
3. cf ssh redcli

You will now be at a shell inside the app instance where our redis-cli is available. To use it to connect to your redis instance, we need to know the DNS name and password of the redis service. If you bound the service to the redcli app, you can see this by typing "export" and hitting enter. You'll see something like:
~~~~
...
declare -x VCAP_APP_PORT="8080"
declare -x VCAP_PLATFORM_OPTIONS="{\"credhub-uri\":\"https://credhub.service.cf.internal:8844\"}"
declare -x VCAP_SERVICES="{\"p.redis\":[{
  \"label\": \"p.redis\",
  \"provider\": null,
  \"plan\": \"cache-small\",
  \"name\": \"cro-redis-cache\",
  \"tags\": [
    \"redis\",
    \"pivotal\",
    \"on-demand\"
  ],
  \"instance_name\": \"cro-redis-cache\",
  \"binding_name\": null,
  \"credentials\": {
    \"host\": \"q-s0.redis-instance.pcfone-xxxxx.service-instance-e890bf2f-ecd8-4378-b1e3-xxxxxxx.bosh\",
    \"password\": \"F4cXXXXXXXXX\",
    \"port\": 6379
  },
  \"syslog_drain_url\": null,
  \"volume_mounts\": [

  ]
}]}"...
~~~~

Find the variable named "VCAP_SERVICES" and locate your redis service instance under that; in the example above, the URL we want is "q-s0.redis-instance.pcfone-xxxxx.service-instance-e890bf2f-ecd8-4378-b1e3-xxxxxxx.bosh" and the password is "F4cXXXXXXXXX". If you have more than one redis service bound to the redcli app, ensure that you're picking the correct instance by verifying that the name ("cro-redis-cache" above) matches your desired service instance name.

Connect to the redis service by doing:
~~~~
/home/vcap/app/redis-cli -h q-s0.redis-instance.pcfone-xxxxx.service-instance-e890bf2f-ecd8-4378-b1e3-xxxxxxx.bosh
~~~~

You'll be presented with a prompt like:
~~~~
q-s0.redis-instance.pcfone-xxxxx.service-instance-e890bf2f-ecd8-4378-b1e3-xxxxxxx.bosh:6379>
~~~~

Before you can issue any commands, you have to authenticate. There is no username, only a password. At the above prompt, issue the following command:
~~~~
AUTH F4cXXXXXXXXX
~~~~
Replace "F4cXXXXXXXXX" with the password you identified in the VCAP_SERVICES environment variable. You can now issue any standard redis commands.

# How to build
You should not need to perform the below process unless the "redis-cli" executable in this repository gives you an error such as 
~~~~
/home/vcap/app/redis-cli: /lib/x86_64-linux-gnu/libm.so.6: version `GLIBC_2.29' not found (required by /home/vcap/app/redis-cli)
/home/vcap/app/redis-cli: /lib/x86_64-linux-gnu/libc.so.6: version `GLIBC_2.28' not found (required by /home/vcap/app/redis-cli)
~~~~
when run inside the PAS application instance, or if you require the CLI for a newer release of redis. In that case, you can rebuild this by following these steps. Start by doing the steps above under "How to deploy" because we need a PAS application instance inside which to do the compilation.
1. cf ssh redcli
2. mkdir redis-src
3. cd redis-src
4. wget https://github.com/antirez/redis/archive/5.0.7.tar.gz
5. tar -zxvf 5.0.7.tar.gz
6. cd redis-5.0.7/
7. make CFLAGS="-static" EXEEXT="-static" LDFLAGS="-I/usr/local/include/"
8. exit

Now the new executable has been built and we need to pull it down to save it so we don't need to redo this every time the redcli app instance restarts. You should have exited the redcli application instance shell and be working on your development machine again. You need three pieces of information to do this - an ssh-code for the redcli app, the guid of the redcli app, and the system domain of your PAS foundation.

Gather the system domain of your PAS foundation by doing
~~~~
cf target
~~~~
This will output a block like
~~~~
CRO-MacBook-Pro:redis-cli-for-PAS colrich$ cf target
api endpoint:   https://api.run.pcfone.io
api version:    2.139.0
user:           colrich@pivotal.io
org:            pivot-colrich
space:          development
~~~~
The system domain is the "api endpoint" minus the "https://api." portion. In the above example, the system domain is "run.pcfone.io".



Gather the redcli app guid by doing
~~~~
cf app redcli --guid
~~~~
You should see output like:
~~~~
d1d735e2-a540-4497-XXXX-433679a0dce8
~~~~

Gather the redcli app ssh-code by doing
~~~~
cf ssh-code redcli
~~~~
This will output a string like:
~~~~
U66V6XXXXX8
~~~~

Note that ssh-codes are one-time use so if you need to rerun the "scp" command below you'll need to generate a fresh ssh-code.

To pull down the redis-cli executable that we built in the redcli application instance, do
~~~~
scp -P 2222 -oUser=cf:GUID/0 ssh.SYSTEMDOMAIN:/home/vcap/redis-src/redis-5.0.7/src/redis-cli ./redis-cli
~~~~
Replace "GUID" with the app guid and "SYSTEMDOMAIN" with your system domain (note that "ssh." is prepended to the system domain to make the ssh endpoint). In my case, I would run
~~~~
scp -P 2222 -oUser=cf:d1d735e2-a540-4497-XXXX-433679a0dce8/0 ssh.run.pcfone.io:/home/vcap/redis-src/redis-5.0.7/src/redis-cli ./redis-cli
~~~~

You'll get a prompt like:
~~~~
cf:d1d735e2-a540-4497-XXXX-433@ssh.run.pcfone.io's password:
~~~~
Enter the ssh-code. The redis-cli that we compiled on the redcli application instance will be downloaded to your local machine. Save this file and use it in place of the one included in this repository.
