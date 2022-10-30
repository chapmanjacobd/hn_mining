# hn_mining

## Observations

### soft quotes in hn_comment_nouns_common

126179 ”

There are 126,000+ quotations on HN using soft quotes. I find this deeply disturbing.

### Intel vs AMD

```sh
$ rg 'Intel|AMD' hn_story_nouns_common
3921 AMD
5477 Intel
```

`ripgrep` recognizes hn_comment_nouns_common as a binary file and I don't know how to fix that so using grep:

```sh
$ grep --binary-files text -E 'Intel|AMD' hn_comment_nouns_common
  49167 AMD
  87458 Intel
```

And Intel is mentioned even more!

### AI vs Blockchain

```sh
$ grep --binary-files text -E 'AI|blockchain' hn_comment_nouns_common
  68368 blockchain
 160277 AI
```

I'd guess 99% of blockchain references are either people complaining about the inefficiencies of blockchain as a database or scammy spams.

### Sun vs Moon

```sh
$ grep --binary-files text -E 'sun|moon' hn_comment_nouns_common
  30905 moon
  36100 Samsung
  44937 sun
```

This is closer than I thought it would be!

```sh
$ grep --binary-files text -E 'Apollo$|Helios$|Ra$' hn_comment_nouns_common
     38 Sun Ra
     49 React+Apollo
    230 The Apollo
    278 Helios
    762 LoRa
   7299 Apollo
```

But I'm more surprised that there are only 0 or 1 mentions of the [sun god Ra](https://en.wikipedia.org/wiki/Solar_deity).

**Feel free to open a PR** to add more observations here

## Watch HackerNews from your CLI

Explore 39 years of content without leaving your teletype machine!

```sh
pip install xklb
wget https://github.com/chapmanjacobd/hn_mining/raw/main/hackernews.tw.db.zst
unzstd hackernews.tw.db.zst
library watch hackernews.tw.db --random
library optimize hackernews.tw.db  # optional: this will build an fts index
```

I had to remove some records so that the file would fit in GitHub :mr_burns.jp2000: See [HN.tw on Github](#hackernews-tubewatch-database-on-github) for details.

The original was uploaded to [zenodo](https://zenodo.org/record/7264173).

NB: The zenodo version does not contain _all_ metadata (subtitles, etc) either. For that, after downloading either the zenodo or GitHub version, you would need to run:

```sh
library tubeupdate --extra hackernews.tw.db # this will likely take several days
```

### Mistakes

If you have a careful eye you will notice that the results are not perfect. Here are some weird things I noticed which are probably my fault:

#### mistakes, hn_story_common_domains

- duplicates (or maybe this is a bug in uniq (GNU coreutils) 9.0 ?)
- co.uk and other two letter subdomains are missing from my awk query

#### mistakes, nouns

- ” is included (fixed with [v1.19.027](https://github.com/chapmanjacobd/library/commit/7a3c313a8a95a5b7b0a8c45c900c1bdfa46e7fbc#diff-f304c25583d5896a197bcf66253cb61730c22a245fc998bb1b07cc9acb977b68)) but I'm too lazy to update the data in this repo unless someone submits a PR here

#### mistakes, tubewatch

- In hindsight I probably should've filtered out some spam by using `is_dead IS NULL` or something

## Recipes

Prep

```sh
pip install xklb aiohttp
library hnadd hn.db -v  # takes a few days to download 34 million records but you can press ctrl+c when your patience catalyzes
```

For reference, it took 22 minutes to catch up to the latest 150,000 comments and stories since the last time I ran it. The file is 24GB now.

I was able to compress the whole database down to 11GB using zstd. You can download it [here via zenodo](https://zenodo.org/record/7263982).

### hn_story_nouns / hn_comment_nouns

```fish
# Exported `SELECT title || ' ' || COALESCE(text, '') text FROM hn_story` via dbeaver
# For hn_comments I used `SELECT text FROM hn_comment`. sqlite3 or sqlite-utils might've been faster but dbeaver was fast enough
cat hn_story_202210242110.csv | library nouns > hn_story_nouns

function asc
    sort | uniq -c | sort -g
end

cat hn_story_nouns | asc > hn_story_nouns_common

sed '/^      2 /Q' hn_story_nouns_common > hn_story_nouns_unique
sed -i -e 1,(cat hn_story_nouns_unique | count)d hn_story_nouns_common
```

### hn_story_common_domains

```fish
function domains
    awk -F \/ '{l=split($3,a,"."); print (a[l-1]=="com"?a[l-2] OFS:X) a[l-1] OFS a[l]}' OFS="." $argv
end

cat hn_story_urls | domains | asc > hn_story_common_domains
sed -i -e 1,(sed '/^      2 /Q' hn_story_common_domains | count)d hn_story_common_domains  # remove boring unique values
```

### hackernews tubewatch database

```fish
sqlite-utils hn.db 'select distinct path from hn_story' --csv > hn_story_urls_d
xsv select path hn_story_urls | grep -v twitter | sponge hn_story_urls  # remove quotes; and twitter because it 429 so easy like a little bitch baby
split -d -C 10MB hn_story_urls hn_story_urls_

function b
    fish -c (string join -- ' ' (string escape -- $argv)) &
end

for f in hn_story_urls_*
    b library tubeadd --playlist-files --no-sanitize --safe examples/hackernews.tw.db $f
end
```

### hackernews tubewatch database on github

JWPlatform, Imgur, CBSNewsEmbed, CNN, Viidea, etc did not seem to work after a few tests:

```sh
lb-dev watch ~/github/xk/hn_mining/hackernews.tw.db -w "ie_key = 'JWPlatform'"
https://www.businessinsider.com/australia-face-scan-porn-home-affairs-internet-2019-10
Player exited with code 2
```

- ie_key WSJ and FoxNews loaded nearly every time but it was always a video that didn't relate to the article lol... NBCNews also suffered from this sometimes
- ie_key Twitter would hang often or not have a video but I did see [this really cool thing](https://twitter.com/hrishiptweets/status/1189172945437904898)!
- ie_key WashingtonPost, CBCPlayer do not save a valid URL id so all of that data was impossible to pipe to mpv; ie_key iheartradio as well to some extent
- ie_key SafariApi just seemed to point to Oreilly books
- holy cow batman CCC and InfoQ videos load fast. They must have a 16 core PC with a 10 bit pipe?

```sql
DELETE FROM media WHERE ie_key IN ('JWPlatform', 'Imgur', 'WSJ', 'Twitter', 'WashingtonPost', 'CBSNewsEmbed', 'CNN', 'SafariApi', 'Viidea', 'NBCNews', 'FoxNews','NineCNineMedia', 'Mixcloud', 'CBCPlayer', 'LinkedIn','AmazonStore', 'Spotify');
DELETE FROM media WHERE playlist_path LIKE '%bostonglobe.com%';
ALTER TABLE media DROP COLUMN tags;
sqlite-utils disable-fts hackernews.tw.db media
sqlite-utils vacuum hackernews.tw.db
zstd hackernews.tw.db
```
