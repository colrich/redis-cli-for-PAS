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
/home/vcap/app/redis-cli -h q-s0.redis-instance.pcfone-xxxxx.service-instance-e890bf2f-ecd8-4378-b1e3-xxxxxxx.bosh

You'll be presented with a prompt like:
  q-s0.redis-instance.pcfone-xxxxx.service-instance-e890bf2f-ecd8-4378-b1e3-xxxxxxx.bosh:6379>

Before you can issue any commands, you have to authenticate. There is no username, only a password. At the above prompt, issue the following command:
  AUTH F4cXXXXXXXXX

Replace "F4cXXXXXXXXX" with the password you identified in the VCAP_SERVICES environment variable. You can now issue any standard redis commands.

