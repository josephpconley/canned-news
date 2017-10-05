# Canned News

This project automatically creates a daily email newsletter from new activity in the RSS feeds you follow. Here's [an example of an email generated and sent by this project](https://archive.canned.news/emails/ryaoL8k9-.html).

## Why?

I like RSS feeds and the [Feedbin Reader](https://feedbin.com/), but I also like getting a daily email summary of my news and blogs. I looked around a bit and didn't find a solution that would send me the latest posts from each of my tags every day, so I set this project up.

## Features

- Uses Feedbin to collect and store articles in a Postgres database.
- Resolves the final URL for each article.
- Summarizes article content using [Aylien's text analysis API](https://developer.aylien.com/).
- Gets the number of social media interactions using the [SharedCount API](https://www.sharedcount.com/).
- Gets an image for each entry.
- Sends a daily email to you using the email from your Feedbin account.

## Development

### Requirements

This project relies on a lot of external services. This allows it to do a lot with very little code, but it's also a decent amount of work to get set up. At some point I'd like to make this easier, but here it is for now:

- Feedbin - The whole project relies on you having a paid Feedbin account
- Sendgrid - Sends emails
- Amazon S3 - Hosts static HTML versions of your emails
- Aylien - Text analysis
- SharedCount - Aggregates social data about posts
- Node/NPM - Setup scripts are run in NPM
- Docker - Used for local development and deployment
- Hyper.sh - Container hosting and cron jobs

### Architecture

This project includes several Node microservices:

- **Collector** (`/collector`) - Responsible for collecting new entries from Feedbin every hour and updating the list of feeds for each of your tags.
- **URL Resolver** (`/url_resolver`) - Resolves any URL redirects from entries for improved extraction.
- **Summarizer** (`/summarizer`) - Summarizes the article content.
- **Image Extractor** (`/image_extractor`) - Extracts the dominant image url for each article.
- **Socializer** (`/socializer`) - Grabs data from SharedCount about social activity on each article.
- **Mailer** (`/mailer`) - Sends an email based on the preferences set for your newsletters in the `tags` table.

You can find bash scripts for Docker in the `/docker` folder and a database schema file in the `/database` directory. There is also a `/shared` directory for shared Node scripts.

### Local Setup

### Prerequisites

Coming soon!

### Creating a Newsletter

Newsletters are currently created manually. You have to add the newsletter to the database, to Feedbin, and create a new list segment in Mailchimp. Eventually I'll get around to automating this process, but for now, here's how it works.

#### 1. Add the RSS feed to Feedbin

Most major news outlets and blogs offer RSS feeds. For example, here are RSS feeds for the following news outlets:

- [Wall Street Journal](http://www.wsj.com/public/page/rss_news_and_feeds.html)
- [New York Times](http://www.nytimes.com/services/xml/rss/index.html)
- [Washington Post](https://www.washingtonpost.com/rss-feeds/2014/08/04/ab6f109a-1bf7-11e4-ae54-0cfe1f974f8a_story.html?utm_term=.bda297015343)
- [The Guardian](https://www.theguardian.com/help/feeds)
- [USA Today](https://www.usatoday.com/rss/)

Add the feeds you want to Feedbin and tag them. This tag name will become the name of the Newsletter.

#### 2. Create a Segment in Mailchimp

All subscribers are currently stored in a single list with a segment for each newsletter. Create a new group for this Newsletter. Title it the same thing as the tag on Feedbin.

#### 3. Add the Newsletter to your Database

Next the application's database needs to know about the new tag. Create a new Newsletter in the database. Be sure to include the segment ID in the database entry.

Next time you run the collector, articles should be retrieved for the new Newsletter.

### Testing

Coming soon!

### Deployment

Since this project is changing frequently, I haven't automated deployments yet. There are some scripts that make it slightly easier though.

- After making updates, build the latest image for all the containers: `npm run app:build`
- Push them to Docker Hub: `npm run app:push`.
- Create a `.env.prod` file for each service.
- Bring up the database container manually (only needs to be done the first time): `npm run db:prod:up`.
- Run any sql files in the `/database` directory.
- Run the deployer: `npm run app:deploy`.

You can then manually run each of the services to test them out:

- Collector: `npm run collector:prod:run`
- URL Resolver: `npm run url-resolver:prod:run`
- Summarizer: `npm run summarizer:prod:run`
- Image Extractor: `npm run image-extractor:prod:run`
- Socializer: `npm run socializer:prod:run`
- Mailer: `npm run mailer:prod:run`

Once you verify they're working, you should use a cron job to automatically run these commands.

Run the collector every hour, getting all entries from Feedbin collected within the past 2 hours:

```bash
hyper cron create --minute=5 --hour=* --name fbm-collector-cron --env-file $(pwd)/collector/.env.prod --link fbm-postgres-1:postgres karllhughes/fbm-collector
```

Run the URL Resolver every hour, a few minutes after the collector, getting all posts published in the past 25 hours:

```bash
hyper cron create --minute=15 --hour=* --name fbm-url-resolver-cron --env-file $(pwd)/url_resolver/.env.prod --link fbm-postgres-1:postgres karllhughes/fbm-url-resolver
```

Run the Summarizer every hour, a few minutes after the URL Resolver, getting all posts published in the past 25 hours:

```bash
hyper cron create --minute=25 --hour=* --name fbm-summarizer-cron --env-file $(pwd)/summarizer/.env.prod --link fbm-postgres-1:postgres karllhughes/fbm-summarizer
```

Run the Image Extractor every hour, a few minutes after the Summarizer, getting all posts published in the past 25 hours:

```bash
hyper cron create --minute=35 --hour=* --name fbm-image-extractor-cron --env-file $(pwd)/image_extractor/.env.prod --link fbm-postgres-1:postgres karllhughes/fbm-image-extractor
```

Run run the Socializer every day at 8:45 UTC:

```bash
hyper cron create --minute=45 --hour=8 --name fbm-socializer-cron --env-file $(pwd)/socializer/.env.prod --link fbm-postgres-1:postgres karllhughes/fbm-socializer
```

Finally, the mailer should be run every day at 9:00 UTC:

```bash
hyper cron create --minute=0 --hour=9 --name fbm-mailer-cron --env-file $(pwd)/mailer/.env.prod --link fbm-postgres-1:postgres karllhughes/fbm-mailer
```

## Contributing

While this project is primary to scratch my own itch, you're welcome to suggest improvements. Just make a PR or create an issue.

## License

Copyright 2017, Karl Hughes

>   Licensed under the Apache License, Version 2.0 (the "License");
>   you may not use this file except in compliance with the License.
>   You may obtain a copy of the License at
>
>     http://www.apache.org/licenses/LICENSE-2.0
>
>   Unless required by applicable law or agreed to in writing, software
>   distributed under the License is distributed on an "AS IS" BASIS,
>   WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
>   See the License for the specific language governing permissions and
>   limitations under the License.
