# vite-config-for-remote-dev
how to setup vite for remote development. 

## The Problem
Laravel project being worked on a remote dev server for shared programming - setting up vite

## My Setup
Not sure if this will be appliable to all setups with remote servers, but here is my setup

1) remote dev server, located in my local network as a dedicated piece of hardware(and acessable through a dedicated IP for whitelisted dev team members)
2) dedicated server has a dedicated IP
3) dedicated server is managed by Laravel Forge (App config)
4) dedciated server is accessed through DNS
5) a `.dev` or any other domain ending to access your dev sites via whitelisted IPs

## How to setup vite with a remote dev server

### Presetup info

1) Laravel forge is connected to your dedicated remote dev server
2) Laravel forge has a site created called `awesome-project.your-company.dev`
3) this laravel forge app contains a instance of your app 


### Within your application project

1) Add a custom script to access a custom `vite.config.js` within your `package.json`
```.js
// in package.json
{
    "private": true,
    "scripts": {
        "remote": "vite --config remote.vite.config.js",
        "dev": "vite",
        "build": "vite build"
    },
    ...

```
2) create `remote.vite.config.js` within your application
```.js
//remote.vite.config.js
import { defineConfig } from 'vite';
import laravel, { refreshPaths } from 'laravel-vite-plugin';
const fs = require('node:fs');

export default defineConfig({
    server: {
        host:'0.0.0.0',    // local access
        port: 5174,        // interal port
        strictPort: true,  
        hmr: {
            protocol: 'wss',                           //super important to avoid https issues
            host: 'awesome-project.your-company.dev',  // domain link to your forge site
            clientPort: 5173,                          // exteral port that will point to nginx
        }
    },
    plugins: [
        laravel({
            input: [
                'resources/css/app.css',
                'resources/js/app.js',
            ],
            refresh: [
                ...refreshPaths,
                'app/Http/Livewire/**',
            ],
        }),
    ],
    build: {
        chunkSizeWarningLimit: 800,
    },
});
```

### Within Laravel Forge

1) use 'lets-encrypt' to create your SSL Certs for the site, by ethier using a whitelisting bot or open your server to the word on ports 80 and 443 then closing. 
2) edit your nginx config for your site `awesome-project.your-company.dev`

The main point here, is your going to add a new `server` block with the following info

```.conf

...
server {
    listen 5173 ssl; #HMR Port from 'remote.vite.config.js'
    server_name awesome-project.your-company.dev; #your host name for the site
    location / {
        proxy_pass http://127.0.0.1:5174/; #port from the server.port from 'remote.vite.config.js'
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection upgrade;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $remote_addr;
    }
    
    # Change the bellow `ssl_certificate` and `ssl_certificate_key` below to match the server block below after you grab new certs
    # FORGE SSL (DO NOT REMOVE!)
    ssl_certificate /etc/nginx/ssl/awesome-project.your-company.dev/CHANGE_ME/server.crt; 
    ssl_certificate_key /etc/nginx/ssl/awesome-project.your-company.dev/CHANGE_ME/server.key;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384;
    ssl_prefer_server_ciphers off;
    ssl_dhparam /etc/nginx/dhparams.pem;
    
}

...
```

### Wrap up

now you can run `npm run remote` which will run `vite` with you remote dev config, and use nginx to proxy it back to the user
