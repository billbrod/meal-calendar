# meal-calendar
Vega project to visualize meals

# Usage

NOTE: This shows meals up to a month before the current date. To disable this
filtering, find the following lines and delete them (as of this writing, they're
from lines 57 to 60 in `calendar.vg.json`):

``` json
                {
                    "type": "filter",
                    "expr": "datum.date < now() - 1000*60*60*24*30"
                },
```

## Data

An example csv has been included at `data/dinner.csv`, which you should replace
with your data (to change the path, change the `"url"` field of `"data"` in
`calendar.vg.json`; this can also be a url, see [vega
docs](https://vega.github.io/vega/docs/data/) for more details).

We use the following columns:
- `date`: date, formatted as `YYYY-MM-DD` (with zero-padding for months and days).
- `type`: what kind of meal. expected values: `cooked, leftovers, frozen,
  restaurant, takeout`.
- `main dish`: name of the main dish.
- `main dish type`: category of main dish (e.g., `stew`, `vegetable`).
- `main dish cuisine`: cuisine of the main dish.
- `main dish source`: category where main dish came from. not constrained, but
  expected to something like `website`, `cookbook`, etc.
- `main dish source detail`: specific source for main dish, such as the name of
  the website, restaurant, or cookbook. Can be empty.
- `main dish source link`: if recipe can be found online, link to it.
- `side dish 1{, type, cuisine, source, source detail, source link}`: same as
  above, but for side dish 1. (Currently, we only have space for one side dish).
  Can be empty.
- `pesctarian`: `yes/no`, whether meal was pescatarian or not.
- `vegetarian`: `yes/no`, whether meal was vegetarian or not.
- `vegan`: `yes/no`, whether meal was vegan or not.
- `non-vegan ingredient`: if meal was not vegan, the ingredients that made it
  so.
- `protein`: main protein for meal. Can be empty.
- `alcohol`: alcohol with meal, if any. Can be empty.

If you want to include a comma in one of your columns, wrap the value in quotes,
e.g., `"butter, eggs"`.

## Data to use for color schemes

In `index.html`, the `data_select` element defines a dropdown menu for selection
of the data column which determines the color filling the squares. The `value`
of each `<option>` must be one of the columns in the `dinner.csv`, the text can
be whatever you want. To add another data visualization option (or remove one),
modify that list. See [below](#color-schemes) for how to determine the color scheme.

## Color Schemes

I use a variety of custom color schemes for the fields, which requires defining the
mapping between values and colors. If you have different values than me, these
probably won't work.

The color maps are defined in `index.html`, the lines with `vega.scheme('NAME',
[COLORS])`, where `'NAME'` is the name of the color scheme and `[COLORS]` is the
list of the color hexcodes.

In `calendar.vg.json`, the `palette` signal gives the mapping between column and
palette: the key must be one of the `value` from the `data_select` element (and
thus, column from the data csv), while the value is a name of the color scheme,
either one of the [included vega
schemes](https://vega.github.io/vega/docs/schemes/) or one of the custom ones
defined in `index.html`.

Below that, the `domain` signal defines the ordering of the values. The order is
the same as for the color scheme, so this is the way to determine which color
goes with which value. For example, if the scheme is `[#000000, #ffffff]` and
the corresponding domain is `['cooked', 'leftovers']`, then all sqaures with
`'cooked'` with have fill `#000000` and those with `'leftovers'` will have
`#ffffff`.

In both, you can use blanks to have empty entries in the legend (though all
entries in domain have to be unique, so you'd want to use different space
characters), which is convenient for controlling the arrangements of legend
elements more carefully.

Thus, whenever you add a new value to your data, you'll need to update your
domain (and possibly your color scheme, depending on how many colors you
defined). If you want to avoid this, just pick one of the vega schemes that you
like and set that in palette (and then remove the corresponding mapping in
`domain`, see `main dish source` for an example).

# Deployment

## Run locally

To run locally, run a ython webserver: `python -m http.server 8002`, then open
up your browser to `localhost:8002`. You can replace the number with some other
port, you just want to make sure it doesn't conflict with something else.

## Deploy on web

In order to deploy this on my website, I just got this onto my server and ran a
python webserver as above, then added the following to my `nginx` config:

``` nginx
location /meal-calendar {
  include proxy_params;
  proxy_pass http://127.0.0.1:8002;
  proxy_ssl_server_name on;
  proxy_set_header Upgrade $http_upgrade;
  proxy_set_header Connection "upgrade";
  proxy_http_version 1.1;
  proxy_set_header X-Forwarded-Proto $scheme;
  proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
  proxy_set_header Host $host:$server_port;
  proxy_buffering off;
}
```

Not all of that is probably necessary, but the `proxy_pass` line definitely is.
Because of how paths work, I needed to run the `python -m http.server 8002`
command from my home directory, one directory above the `meal-calendar` repo.

### Keeping data up-to-date

You can manually update the data on your server, but I keep my data in a private
github repo and figured there was a fancier way to keep them synchronized:
webhooks. My data lives in a `meals` directory on my server, then I followed
[this
guide](https://maximorlov.com/automated-deployments-from-github-with-webhook/),
with the following `hooks.json`, using a small bash script that contains `git
pull; cp dinner.csv ../meal-calendar/data/dinner.csv` for the
`"execute-command"`. Hardlinks don't work with git pull and symlinks don't work
with however vega grabs the data, so copying is necesary.
