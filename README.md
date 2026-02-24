# backstage-vm

### nginx config
sudo nginx -t # Test Nginx config for errors
sudo systemctl restart nginx

sudo nano /etc/nginx/sites-available/backstage

### Docker Image
https://backstage.io/docs/deployment/docker

```
yarn install --immutable

# tsc outputs type definitions to dist-types/ in the repo root, which are then consumed by the build
yarn tsc

# Build the backend, which bundles it all up into the packages/backend/dist folder.
yarn build:backend
```

```
docker image build . -f packages/backend/Dockerfile --tag backstage

docker run -it -p 7007:7007 backstage
```
