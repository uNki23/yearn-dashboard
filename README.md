# yearn-dashboard

Inspired by https://github.com/yieldfarming/yieldfarming 

I just wanted to only see my holdings / tokens and not in USD but EUR.

Quick and dirty :)

## Personal setup

Adjust your Wallet address in ./helpers/constants.js -> MY_ADDRESS

Adjust your investments / tokens under ./App.vue script in vaultCompatibleTokens -> don't forget to set initial amount and timestamp of investment. I used https://www.epochconverter.com/ for the conversion.

## Project setup
```
npm install
```

### Compiles and hot-reloads for development
```
npm run serve
```

### Compiles and minifies for production
```
npm run build
```

### Lints and fixes files
```
npm run lint
```

### Customize configuration
See [Configuration Reference](https://cli.vuejs.org/config/).
