---
title: Migrating from Pinboard to Linkding (Part 1)
date: "2023-07-16"
publishdate: "2023-07-16"
categories:
- Pinboard-to-Linkding Migration
tags:
- pinboard
- linkding
- bookmarks
- self-hosted
---
I recently migrated my bookmarks from Pinboard to Linkding. There were a number of factors that led me to make this move, as well as some cool technical challenges I solved along the way. In part 1 of this series, I'll talk about why I switched & some of the tooling I created to automate the migration.

## Reasons for Migrating

### Why Use Linkding?
[Linkding](https://github.com/sissbruecker/linkding) is an open-source, self-hosted bookmark manager with excellent support for Docker deployment. It's written using the Django web framework in Python and includes a well-designed REST API. Having the code available gives me some peace of mind--if the creator decides to stop supporting the project, I'm free to keep my server running & fork the source code.

Many of the features that I love in Pinboard (e.g. tagging, archiving) are either directly supported or have similar functionality. For archiving, Linkding supports sending your pages to the [Internet Archive](https://archive.org/). This doesn't allow "local" storage of the archived sites, but does support Wayback Machine lookups if the site had been previously archived.

### Why Leave Pinboard?
I've experienced a number of service degradations in the past few months, including a site outage and bugs in the Chrome extension. I also experienced an issue with archiving (a paid feature) earlier this year. An attempt to reach the support team via email went unanswered, though the feature was eventually brought back online.

It's evident that the maintainer & his team are either too overwhelmed to keep up with maintenance, or are satisfied with the performance & feature set of the site (they are called Nine Fives software, after all). Relying on a closed-source application for my bookmarking was already a risky proposition, but the instability of Pinboard led me to look at alternatives.

## Technical Requirements & Solutions

### Getting Linkding Set Up
I've settled on a Docker + reverse proxy setup for my self-hosted services. I run most of these on a DigitalOcean droplet, though I have a homelab for things I don't want to expose to the internet. This warrants another post, but here's a quick summary:

1. Set up an A record for your new app
2. On the droplet, create the container infrastructure:
    - `docker-compose.yml` (Linkding includes an [example file](https://github.com/sissbruecker/linkding/blob/master/docker-compose.yml))
    - A systemd service file
    - An entry in the reverse proxy's config
3. Start up the service
4. Generate an SSL certificate
5. Confirm operation!

This should leave you with a functioning server hosted on your domain with HTTPS support. Once the service is up & running we can get started on importing our bookmarks into Linkding.

### Pinboard Export Format
To start, I downloaded a JSON dump of my Pinboard bookmarks. This gives us an array of objects representing each of the bookmarks:
```json
[
  {
    "href": "https://davidgraeber.org/articles/against-economics/",
    "description": "Against Economics – David Graeber",
    "extended": "",
    "meta": "28310828d1a7a16f7e86533bb2038349",
    "hash": "fc1397fdccf724b2de01e932fd57a334",
    "time": "2023-06-03T22:22:41Z",
    "shared": "no",
    "toread": "no",
    "tags": "economics anarchism"
  },
  // snip
]
```

I care most about the `href` property (this should go without saying :)), as well as the `description` and `tags` fields. The `time` field represents the time I originally created the bookmark; more on that in a bit!

### Pinboard-to-Linkding Translation Logic
Linkding does support a bulk-upload feature, however (at time of migration) it did not support tag imports. This is a key feature for my use case, so I needed to roll my own import logic.

I decided to use Python to transform the Pinboard-exported objects to Linkding-compatible objects. Mostly this was translating field names, though I also had to change some string fields (e.g. `"shared": string` becomes `"shared": bool`. To make things tidy, I created a `PinboardEntry` class that encapsulated the translation logic:
```python
class PinboardEntry:
    @staticmethod
    def parse_tags(tag_str):
        if not tag_str:
            return []
        return tag_str.split(" ")

    @staticmethod
    def parse_time(time_str):
        return datetime.datetime.strptime(time_str,TIME_SPEC)

    @staticmethod
    def parse_bool(bool_str):
        return bool_str == "yes"

    def __init__(self, href, description, meta, hash, shared, to_read, time,
                 tags, extended):
        self.url = href
        self.title = description
        self.description = extended

        self.time = PinboardEntry.parse_time(time)

        self.shared = PinboardEntry.parse_bool(shared)
        self.to_read = PinboardEntry.parse_bool(to_read)

        self.tags = PinboardEntry.parse_tags(tags)

        self.meta = meta
        self.hash = hash
```

I also added a `classmethod` to create a new `PinboardEntry` from the result of Python's `json.loads` function. This allows for only specific arguments to be sent to the constructor, though it may be a bit too much abstraction:

```python
@classmethod
def from_json(cls, json_obj):
    args = [
        json_obj["href"],
        json_obj["description"],
        json_obj["meta"],
        json_obj["hash"],
        json_obj["shared"],
        json_obj["toread"],
        json_obj["time"],
        json_obj["tags"],
        json_obj["extended"],
    ]
    return cls(*args)
```

Finally, we add a `to_linkding_format` method to get a `dict` with Linkding's required parameters. I added a stringified version of the extra keys from Pinboard just in case we could make use of them in the future:

```python
@property
def notes(self):
    d = {
        "meta": self.meta,
        "hash": self.hash,
        "time": self.time.strftime(TIME_SPEC),
    }

    note_lines = [
        "IMPORTED FROM PINBOARD",
        json.dumps(d, sort_keys=True, indent=2),
    ]

    return "\n".join(note_lines)

def to_linkding_format(self):
    res = {
        "url": self.url,
        "title": self.title,
        "description": self.description,
        "notes": self.notes,
        "shared": self.shared,
        "date_added": self.time.isoformat(),
        # Set these to default values.
        "unread": True,
        "is_archived": False,
    }
    # Only add this key if we have `tags`; I saw some 400 errors with empty values.
    if self.tags:
        res["tag_names"] = self.tags

    return res
```

### Making API Calls

At this point we have the basic infrastructure required to parse the export & do some data transformation. The remainder is sending HTTP requests to the appropriate endpoints. I used the [grequests library](https://github.com/spyoungtech/grequests) to allow for more performance.

```python
# Turn JSON entries into `PinboardEntry` objects
json_text = read_file(PINBOARD_EXPORT_FILE_NAME)
json_objs = json.loads(json_text)

pinboard_entries = [PinboardEntry.from_json(json_obj) for json_obj in json_objs]

# Send each `PinboardEntry` to the Linkding API
rs = (
    grequests.post(
        LINKDING_ENDPOINT_URL,
        data=pinboard_entry.to_linkding_format(),
        headers=auth_header,
    )
    for pinboard_entry in pinboard_entries
)

grequests.map(rs, size=100)
```

The code for v1 of this script can be found in [this GitHub gist](https://gist.github.com/jls83/ffc9aae2d5cbe9fb68f5dd5e9443ad79). You'll need to pass some environment variables for your Linkding API token & the Pinboard export file.

## Part 1 Conclusion

At this point we have a nice solution for getting the Pinboard bookmarks into Linkding. However, there is one key feature that I needed: In part 2 of this series, I'll talk about setting the `date_added` field on bookmarks created via the API so that our original creation order is preserved. Thanks for reading!
