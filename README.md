# hn_mining

## Observations


## Recipes

Prep

```sh
pip install xklb aiohttp
library hnadd hn.db -v  # takes a few days to download 34 million records but you can press ctrl+c when your patience catalyzes
```

For reference, it took 22 minutes to catch up to the latest 150,000 comments and stories since the last time I ran it. The file is 24GB now.

### hn_story_nouns

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
xsv select path hn_story_urls | sponge hn_story_urls  # remove quotes
split -d -C 10MB hn_story_urls hn_story_urls_

function b
    fish -c (string join -- ' ' (string escape -- $argv)) &
end

for f in hn_story_urls_*
    b library tubeadd --no-sanitize --safe --playlist-files examples/hackernews.tw.db $f
end
```
