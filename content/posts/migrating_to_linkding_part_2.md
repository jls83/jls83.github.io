---
title: Migrating from Pinboard to Linkding (Part 2)
date: "2023-10-16"
draft: false
categories:
- Pinboard-to-Linkding Migration
tags:
- pinboard
- linkding
- bookmarks
- self-hosted
---
In [part 1]( {{< relref "posts/migrating_to_linkding_part_1" >}} ) of this series I talked about some reasons why I moved from Pinboard to Linkding, as well as some of the migration steps. In this part I'll talk about some extra requirements I had that necessitated some modifications to the Linkding source code.

Without realizing it, I had come to rely on the creation order of my Pinboard bookmarks as a form of "autobiographical" indexing. I loved seeing how my interests changed (or didn't!) over time, and how my learning process unfolded. Unfortunately, my solution from part 1 of this series did not preserve the "creation time" value of the entries. Linkding marks the `date_added` field as read-only; a choice that absolutely makes sense for the normal use case, but breaks my autobiographical index. Thankfully, my desired behavior is easy to patch in!

## Modifying the Serializer & Service
The Django-side change is thankfully quite small. You can see the change [in my Linkding fork](https://github.com/jls83/linkding/commit/591bfcdedbf78207df288f45018bf7e7b58b61c4), but to summarize:

* Remove `date_added` from the `read_only_fields` for the `BookmarkSerializer` class
* Read `date_added` off of the `validated_data`, with a default value of `None`
* Add logic in the `create_bookmark` service function to handle the case where we do not provide a `date_added`, rather than just using `now()` as the value

Since we already send the `date_added` field in the `PinboardEntry.to_linkding_format` method, we don't need to make any changes to the export script from part 1.

## Running Linkding Locally, Importing Into Prod
There is another hurdle, however. Because my prod instance is running via Docker, I would need to work with Docker to get my modified branch into a container. Not that this is particularly "hard", but it seemed like a bit more effort than I wanted. Additionally, I did not want to run my modified code in production beyond the initial bookmark import.

Instead, I decided to run Linkding locally and make whatever modifications I need there. I could then then run the HTTP requests against the local/dev environment. Thankfully the project includes [helpful instructions](https://github.com/sissbruecker/linkding#development) on getting an environment set up locally.

After running the script against my local instance, I now had all of my Pinboard bookmarks imported with their proper creation dates! I just needed to rsync the database file up to my prod server:

```sh
rsync -azP /path/to/local_db.sqlite3 jls83@linkding.prod.server:/home/jls83/local_db.sqlite3
```

Then swap the files & restart the service:
```sh
systemctl stop linkding-docker-compose.service

# Always backup your stuff :)
cp /opt/linkding/data/db.sqlite3 ~/linkding_backup.sqlite3

sudo cp ~/local_db.sqlite3 /opt/linkding/data/db.sqlite3

systemctl start linkding-docker-compose.service
systemctl restart nginx
```

## Conclusion
I now have my "autobiographical" index back, and can keep growing it! I've nearly doubled my bookmark count since I migrated, mostly due to how much easier it is to create bookmarks via the iOS share sheet. I hope these articles help out folks who are looking to make the switch. Thanks for reading!
