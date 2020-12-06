# Managing the Lisk Core process with PM2

This guide is intended to be used a follow up to Punkrock's [Betanet v5 tutorial](https://punkrock.github.io/lisk-betanet-v5-tutorial.html). 

## Basic quick guide 

#### Install PM2  
`npm i -g pm2`

#### Create a custom pm2 start configuration file
`nano pm2.conf.json`

#### Copy paste the following:
```
{
  "name": "lisk-core",
  "script": "lisk-core start" ,
  "env": {
    "LISK_CONFIG_FILE": "lisk-core/config.json",
    "LISK_NETWORK": "betanet",
  }
}
```

Save and exit

#### Start process
`pm2 start pm2.conf.json`

#### Display logs
`pm2 logs`

#### Add processes to startup
```
pm2 save
pm2 startup
```

Execute the generated command

## Advanced configuration

The benefit of using a dedicated PM2 configuration file is that you can easily customize more advanced settings. 

For example, the following config changes the default ports and enables the HTTP plugin (for localhost):

```
{
  "name": "lisk-core",
  "script": "lisk-core start --enable-http-api-plugin" ,
  "env": {
    "LISK_CONFIG_FILE": "lisk-core/config.json",
    "LISK_NETWORK": "betanet",
    "LISK_PORT": 6001,
    "LISK_HTTP_API_PLUGIN_PORT": 6000,
    "LISK_HTTP_API_PLUGIN_WHITELIST": "127.0.0.1"
  }
}
```

