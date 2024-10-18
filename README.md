


# create-svelte



## Developing

```bash
npm install
npm run dev

# or start the server and open the app in a new browser tab
npm run dev -- --open
```

## Building

To create a production version of the app:

```bash
npm run build

# Or preview it with
npm run preview
```

### Docker build

```bash
# For building on Mac with M1/2/3 chip include --platform linux/amd64 to create an amd build
docker build --platform linux/amd64 . -t stensdev-site

# To store docker image as tarball 
docker save stensdev-site > stensdev-site.tar
```

### Publish

```bash
npx nsite-cli@0.1.9 upload . --purge -v -k nsec...  
```


