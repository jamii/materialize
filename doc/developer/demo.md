# Demo instructions

The following demonstrations assume that you've managed to install the Confluent
Platform installed and have started ZooKeeper, Kafka, and the Confluent Schema
Registry on their default ports. You can find setup instructions in the
[Developer guide](develop.md).

## Demo initialization

There are several demos that we can run. The instructions for all of these
assume that you are running commands from the root of the Materialize repo.

Each demo involves starting several terminal sessions:

* A Materialize daemon -- where the magic happens
* A Materialize shell -- the human interface to the materialize engine
    - Requires `psql` (`brew install postgres`)
* At least one Avro producer window -- the channel through which content gets
  into Materialize

The first thing that we need to do is make sure that the core services our demos
rely upon are running. For now, that means we need `confluent connect` and
`materialized`.

We have a collection of helpful tools in `doc/developer/assets/demo/utils.sh`,
including one that makes sure that `confluent` is running and Materialize is up
to date and running.

So, to get the core services running we just run `mtrlz-start`:

```console
$ source doc/developer/assets/demo/utils.sh
$ mtrlz-start
... snip confluent noise ...
... snip cargo noise ...
materialized listening on 0.0.0.0:6875...
```

After we've done that, we can run each demo independently, without needing to
restart `materialized`.

## Basic demo

To get data into materialized we run the `mtrlz-produce` function, which allows
us to interactively put JSON blobs into Materialize, according to a schema.

```console
$ source doc/developer/assets/demo/utils.sh
$ mtrlz-produce quotes '{
    "type": "record",
    "name": "envelope",
    "fields": [
        {
            "name": "before",
            "type": [
                "null",
                {
                    "type": "record",
                    "name": "row",
                    "fields": [{"name": "quote", "type": "string"}]
                }
            ]
        },
        {
            "name": "after",
            "type": [
                "null",
                "row"
            ]
        }
    ]
}'
🚀 You are now in the avro console shell, enter your json events:
```
Now you won't have a prompt. Just enter this manually:
```
{"before": null, "after": {"row": {"quote": "Syntax highlighting is juvenile. —Rob Pike"}}}
{"before": null, "after": {"row": {"quote": "Arrogance in computer science is measured in nano-Dijkstras. —Alan Kay"}}}
{"before": null, "after": null}  # send an empty record to flush
```

Now we are ready to interact with Materialize!

Open another terminal session and start the Materialize shell:

```sql
$ source doc/developer/assets/demo/utils.sh
$ mtrlz-shell
> CREATE SOURCE quotes FROM 'kafka://localhost/quotes' USING SCHEMA REGISTRY 'http://localhost:8081';
> SHOW COLUMNS FROM quotes;
> PEEK quotes;
> SELECT quote, 42 FROM quotes;
> CREATE MATERIALIZED VIEW business_insights AS SELECT quote, 42 FROM quotes;
> PEEK business_insights;
```

Though this doesn't really demonstrate the capabilities of Materialize––it's
just a toy example––`PEEK` is much faster than the corresponding `SELECT`.
However, it is worth noting one can query data in Materialize with arbitrary
`SELECT`s; you aren't limited to _only_ creating materialized views.

## Aggregate demo

For your avro producer session do:

```console
$ source doc/developer/assets/demo/utils.sh
$ mtrlz-produce aggdata '{
    "type": "record",
    "name": "envelope",
    "fields": [
        {
            "name": "before",
            "type": [
                "null",
                {
                    "type": "record",
                    "name": "row",
                    "fields": [{"name": "a", "type": "long"}, {"name": "b", "type": "long"}]
                }
            ]
        },
        {
            "name": "after",
            "type": [
                "null",
                "row"
            ]
        }
    ]
}'
🚀 You are now in the avro console shell, enter your json events:
```
Now you won't have a prompt. Just enter this manually:
```json
{"before": null, "after": {"row": {"a": 1, "b": 1}}}
{"before": null, "after": {"row": {"a": 2, "b": 1}}}
{"before": null, "after": {"row": {"a": 3, "b": 1}}}
{"before": null, "after": {"row": {"a": 1, "b": 2}}}
{"before": null, "after": null}
```

Then, in another session, open the Materialize shell:

```sql
$ source doc/developer/assets/demo/utils.sh
$ mtrlz-shell
> CREATE SOURCE aggdata FROM 'kafka://localhost/aggdata' USING SCHEMA REGISTRY 'http://localhost:8081';
> CREATE MATERIALIZED VIEW aggtest AS SELECT sum(a) FROM aggdata GROUP BY b;
> PEEK aggtest;
```

## Join demo

```console
$ source doc/developer/assets/demo/utils.sh
$ mtrlz-produce src1 '{
    "type": "record",
    "name": "envelope",
    "fields": [
        {
            "name": "before",
            "type": [
                "null",
                {
                    "type": "record",
                    "name": "row",
                    "fields": [{"name": "a", "type": "long"}, {"name": "b", "type": "long"}]
                }
            ]
        },
        {
            "name": "after",
            "type": [
                "null",
                "row"
            ]
        }
    ]
}'
🚀 You are now in the avro console shell, enter your json events:
```
Now you won't have a prompt, just enter this manually:
```json
{"before": null, "after": {"row": {"a": 1, "b": 1}}}
{"before": null, "after": {"row": {"a": 2, "b": 1}}}
{"before": null, "after": {"row": {"a": 1, "b": 2}}}
{"before": null, "after": {"row": {"a": 1, "b": 3}}}
```

Open another terminal and start another producer:

```console
$ source doc/developer/assets/demo/utils.sh
$ mtrlz-produce src2 '{
    "type": "record",
    "name": "envelope",
    "fields": [
        {
            "name": "before",
            "type": [
                "null",
                {
                    "type": "record",
                    "name": "row",
                    "fields": [{"name": "c", "type": "long"}, {"name": "d", "type": "long"}]
                }
            ]
        },
        {
            "name": "after",
            "type": [
                "null",
                "row"
            ]
        }
    ]
}'
🚀 You are now in the avro console shell, enter your json events:
```
Now you won't have a prompt. Just enter this manually:
```json
{"before": null, "after": {"row": {"c": 1, "d": 1}}}
{"before": null, "after": {"row": {"c": 1, "d": 2}}}
{"before": null, "after": {"row": {"c": 1, "d": 3}}}
{"before": null, "after": {"row": {"c": 3, "d": 1}}}
{"before": null, "after": null}
```

Go back to the `src1` terminal and flush the data:

```json
...
{"before": null, "after": null}
```

Then, in another session, open the Materialize shell:

```sql
$ source doc/developer/assets/demo/utils.sh
$ mtrlz-shell
> CREATE SOURCE src1 FROM 'kafka://localhost/src1' USING SCHEMA REGISTRY 'http://localhost:8081';
> CREATE SOURCE src2 FROM 'kafka://localhost/src2' USING SCHEMA REGISTRY 'http://localhost:8081';
> CREATE MATERIALIZED VIEW jointest AS SELECT a, b, d FROM src1 JOIN src2 ON c = b;
> PEEK jointest;
```