# cdk-hugo-pipeline
[![cdk-constructs: experimental](https://img.shields.io/badge/cdk--constructs-experimental-yellow.svg)](https://constructs.dev/packages/@mavogel/cdk-hugo-pipeline)
[![npm version](https://img.shields.io/npm/v/@mavogel/cdk-hugo-pipeline)](https://www.npmjs.com/package/@mavogel/cdk-hugo-pipeline)

This is an AWS CDK Construct for deploying Hugo Static websites to AWS S3 behind SSL/Cloudfront with `cdk-pipelines`, having an all-in-one infrastructure-as-code deployment on AWS, meaning

- self-contained, all resources should be on AWS
- a blog with `hugo` and a nice theme (in my opinion)
- using `cdk` and [cdk-pipelines](https://docs.aws.amazon.com/cdk/v2/guide/cdk_pipeline.html) running
- a monorepo with all the code components
- with a development stage on a `dev.your-domain.com` subdomain

Take a look at the blog post [My blog with hugo - all on AWS](https://manuel-vogel.de/post/2023-04-16-hugo-all-on-aws/) in which I write
about all the details and learnings.

## Prerequisites
1. binaries
```sh
brew install node@16 hugo docker
```
2. a `Route53 Hosted Zone` for `your-domain.com` in the AWS account you deploy into.

If you use [hugo modules](https://gohugo.io/hugo-modules/) add them as git submodules in the `themes` directory, so they can be pulled by the same git command in the `codepipeline`.

## Usage
In this demo case, we will use the `blist` theme: https://github.com/apvarun/blist-hugo-theme, however you can use any other hugo theme. Note, that you need to adapt the branch of the theme you use.

### With a projen template (recommended)
and the [blist](https://github.com/apvarun/blist-hugo-theme) theme.
```sh
mkdir my-blog && cd my-blog

npx projen new \
    --from @mavogel/projen-cdk-hugo-pipeline@~0 \
    --domain your-domain.com \
    --projenrc-ts

npm --prefix blog install
# and start the development server on http://localhost:1313
npm run dev
```


### By hand (more flexible)
<details>
  <summary>Click me</summary>
  
#### Set up the repository
```sh
# create the surrounding cdk-app
npx projen new awscdk-app-ts
# add the desired hugo template into the 'blog' folder
git submodule add https://github.com/apvarun/blist-hugo-theme.git blog/themes/blist
# add fixed version to hugo template in the .gitmodules file
git submodule set-branch --branch v2.1.0 blog/themes/blist
```
#### Configure the repository 
depending on the theme you use (here [blist](https://github.com/apvarun/blist-hugo-theme))
1. copy the example site
```sh
cp -r blog/themes/blist/exampleSite/*  blog/
```
2. fix the config URLs as we need 2 stages: development & production. **Note**: internally the modules has the convention of a `public-development` & `public-production` output folder for the hugo build.
```sh
# create the directories
mkdir -p blog/config/_default blog/config/development blog/config/production
# and move the standard config in the _default folder
mv blog/config.toml blog/config/_default/config.toml
```
3. adapt the config files
```sh
## file: blog/config/development/config.toml
cat << EOF > blog/config/development/config.toml
baseurl = "https://dev.your-domain.com"
publishDir = "public-development"
EOF

cat << EOF > blog/config/production/config.toml
## file: blog/config/production/config.toml
baseurl = "https://your-domain.com"
publishDir = "public-production"
EOF
```
4. ignore the output folders in the file `blog/.gitignore`
```sh
cat << EOF >> blog/.gitignore
public-*
resources/_gen
node_modules
.DS_Store
.hugo_build.lock
EOF
```
5. additionally copy `package.jsons`. **Note**: this depends on your theme
```sh
cp blog/themes/blist/package.json blog/package.json
cp blog/themes/blist/package-lock.json blog/package-lock.json
```
6. *Optional*: add the script to the `.projenrc.ts`. **Note**: the command depends on your theme as well
```ts
project.addScripts({
  dev: 'npm --prefix blog run start',
  # below is the general commands
  # dev: 'cd blog && hugo server --watch --buildFuture --cleanDestinationDir --disableFastRender',
});
```
and update the project via the following command
```sh
npm run projen
```
#### Use Typescript and deploy to your AWS account
Add this to the the `main.ts` file
```ts
import { App, Stack, StackProps } from 'aws-cdk-lib';
import { HugoPipeline } from '@mavogel/cdk-hugo-pipeline';

export class MyStack extends Stack {
  constructor(scope: Construct, id: string, props?: StackProps) {
    super(scope, id, props);

    // we only need 1 stack as it creates dev and prod stage in the pipeline
    new HugoPipeline(this, 'my-blog', {
      name: 'my-blog',
      domainName: 'your-domain.com', // <- adapt here
      siteSubDomain: 'dev',
      hugoProjectPath: '../../../../blog',
    });
}
```
and adapt the `main.test.ts` (yes, known issue. See #28)

```ts
test('Snapshot', () => {
  expect(true).toBe(true);
});
```

which has a `Route53 Hosted Zone` for `your-domain.com`:
</details>

### Deploy it
```sh
# build it locally via
npm run build
# deploy the repository and the pipeline once via
npm run deploy
```
1. This will create the `codecommit` repository and the `codepipeline`. The pipeline will fail first, so now commit the code.
```sh
# add the remote, e.g. via GRPC http
git remote add origin codecommit::<aws-region>://your-blog
# rename the branch to master (wlll fix this)
git branch -m master main
# push the code
git push origin master
```
2. ... wait until the pipeline has deployed to the `dev stage`, go to your url `dev.your-comain.com`, enter the basic auth credentials (default: `john:doe`) and look at you beautiful blog :tada:

## Known issues
- If with `npm test` you get the error `docker exited with status 1`, 
  - then clean the docker layers and re-run the tests via `docker system prune -f`
  - and if it happens in `codebuild`, re-run the build
## Open todos
- [ ] fix this relative path `hugoProjectPath: '../../../blog'`
- [ ] fix the testing issue with the R53 lookup (see #28)
- [ ] a local development possibility in `docker`

## Resources / Inspiration
- [cdk-hugo-deploy](https://github.com/maafk/cdk-hugo-deploy): however here you need to build the static site with `hugo` before locally
- [CDK-SPA-Deploy](https://github.com/nideveloper/CDK-SPA-Deploy/tree/master): same as above
