# hn_mining

Welcome! The goal of this repo is to learn new things from HackerNews without having to read everything.

## Observations

### soft quotes in hn_comment_nouns_common

```sh
$ cat hn_comment_nouns_common
126179 ”
```

There are 126,000+ quotations on HackerNews using smart quotes. I find this deeply disturbing.

### Intel vs AMD

First we look at hn stories (the posts that people comment on but not the comments themselves). Stories include titles and text (what reddit calls [selftext](https://news.ycombinator.com/item?id=20453120)).

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

And Intel is mentioned even more in the comments (probably because AMD makes programmers' lives easier so there is no need to complain)!

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

Feel free to **dig into the data** and open a PR: edit this README to add more observations

## Watch HackerNews from your CLI

Explore 39 years of content without leaving your teletype machine!

```sh
pip install xklb
wget https://github.com/chapmanjacobd/hn_mining/raw/main/hackernews_only_direct.tw.db

library watch hackernews_only_direct.tw.db --random --ignore-errors
```

```sh
$ lb pl hackernews.tw.db -a
╒════════════════════════╤═════════════════╤═════════════════════════════════╤═══════════════════╤════════════════╕
│ path                   │ duration        │ avg_playlist_duration           │   playlists_count │   videos_count │
╞════════════════════════╪═════════════════╪═════════════════════════════════╪═══════════════════╪════════════════╡
│ Aggregate of playlists │ 39 years, 2     │ 4 days, 14 hours and 58 minutes │              3098 │         741500 │
│                        │ months, 27 days │                                 │                   │                │
│                        │ and 20 hours    │                                 │                   │                │
╘════════════════════════╧═════════════════╧═════════════════════════════════╧═══════════════════╧════════════════╛
```

This is what I mean by 39 years of content. 39 years of video running 24/7 (not including 62,876 videos [~8%] where duration is unknown).

```sh
$ lb wt hackernews_only_direct.tw.db -pa
╒═══════════╤═════════════════╤══════════════════════════╤════════╤═════════╕
│ path      │ duration        │ avg_duration             │ size   │   count │
╞═══════════╪═════════════════╪══════════════════════════╪════════╪═════════╡
│ Aggregate │ 18 years, 3     │ 1 hour and 22.92 minutes │        │  115987 │
│           │ months, 17 days │                          │        │         │
│           │ and 22 hours    │                          │        │         │
╘═══════════╧═════════════════╧══════════════════════════╧════════╧═════════╛
```

`hackernews_only_direct.tw.db` is a subset (including only direct URLs; excluding playlist URLs). It is a bit smaller but still indexes over 18 years of content. edit: only 5 years of content with >7 hn score

### Zenodo vs GitHub TubeWatch database

I had to remove some records so that the file would fit in GitHub. See [HN.tw on Github](#hackernews-tubewatch-database-on-github) for technical details.

The original was uploaded to [zenodo](https://zenodo.org/record/7264173).

NB: The zenodo version does not contain _all_ metadata (subtitles, etc) either. For that, after downloading either the zenodo or GitHub version, you would need to run:

```sh
library tubeupdate --extra hackernews.tw.db  # this will likely take several days
library optimize hackernews.tw.db  # optional: this will build an fts index
```

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
- ie_key Twitter would hang often or not have a video but I did see this really cool thing! edit: link removed lol. the URL pointed to something else. same problem as above.
- ie_key WashingtonPost, CBCPlayer do not save a valid URL id so all of that data was impossible to pipe to mpv; ie_key iheartradio as well to some extent
- ie_key SafariApi just seemed to point to Oreilly books
- holy cow batman CCC and InfoQ videos load fast. They must have a 16 core PC with a 10 bit pipe?

```sql
DELETE FROM media WHERE ie_key IN ('JWPlatform', 'Imgur', 'WSJ', 'Twitter', 'WashingtonPost', 'CBSNewsEmbed', 'CNN', 'SafariApi', 'Viidea', 'NBCNews', 'FoxNews','NineCNineMedia', 'Mixcloud', 'CBCPlayer', 'LinkedIn','AmazonStore', 'Spotify');
DELETE FROM media WHERE playlist_path LIKE '%bostonglobe.com%';
ALTER TABLE media DROP COLUMN tags;
```

```sh
sqlite-utils disable-fts hackernews.tw.db media
sqlite-utils vacuum hackernews.tw.db
zstd hackernews.tw.db
```

### Recipe hackernews_only_direct.tw.db

```sh
cp hackernews.tw.db hackernews_only_direct.tw.db
sqlite-utils hackernews_only_direct.tw.db 'delete from media where playlist_path in (select path from playlists)'
```

```sql
sqlite3
ATTACH 'hackernews_only_direct.tw.db' AS h;
ATTACH '/home/xk/lb/hn.db' AS hn;

create table hn_score as
  select path, max(score) as score
  from hn_story
  where path in (select path from h.media)
  group by 1
;

alter table h.media add column score int;

UPDATE
  h.media SET score = (
        SELECT score
        FROM hn_score
        WHERE hn_score.path = h.media.path
    )
  WHERE
      EXISTS(
          SELECT 1
          FROM hn_score
          WHERE hn_score.path = h.media.path
      );  -- this takes about eight minutes
```

## Mistakes

If you have a careful eye you will notice that the results are not perfect. Here are some weird things I noticed which are probably my fault:

### mistakes, hn_story_common_domains

- duplicates (or maybe this is a bug in uniq (GNU coreutils) 9.0 ?)
- co.uk and other two letter subdomains are missing from my awk query

### mistakes, nouns

- ” is included (fixed with [v1.19.027](https://github.com/chapmanjacobd/library/commit/7a3c313a8a95a5b7b0a8c45c900c1bdfa46e7fbc#diff-f304c25583d5896a197bcf66253cb61730c22a245fc998bb1b07cc9acb977b68)) but I'm too lazy to update the data in this repo unless someone submits a PR here

### mistakes, tubewatch

- In hindsight I probably should've filtered out some spam by using `is_dead IS NULL` or something
