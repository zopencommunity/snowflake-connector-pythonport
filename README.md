# snowflake-connector-python for z/OS

The [Snowflake Connector for Python](https://github.com/snowflakedb/snowflake-connector-python),
ported to z/OS as a pure-Python wheel.

DB-API 2.0, key-pair and password authentication, and the full cursor surface
all work. Query results come back as JSON rather than Arrow — see
[Result format](#result-format), which is the one behavioural difference from
the connector on any other platform.

## Installing

With zopen:

```sh
zopen install snowflake-connector-python
```

With pip, the normal zopen way:

```sh
export PIP_EXTRA_INDEX_URL="https://repo.zopen.community/pypi/wheels/simple/"
export PIP_CONSTRAINT="https://repo.zopen.community/pulp/content/constraints/zopen-constraints.txt"
pip install snowflake-connector-python
```

Both variables matter. `cryptography` needs `cffi`, there is no buildable
`cffi` on z/OS — libffi has no XPLINK backend — and the only z/OS `cffi` is the
prebuilt wheel on the zopen index. PyPI always carries a newer `cffi`, pip
resolves by version across both indexes, and the newer one resolves to an sdist
that dies on a missing `ffi.h`:

```
src/c/_cffi_backend.c:15:10: fatal error: 'ffi.h' file not found
```

The constraints file pins `cffi` to the version the index serves, which stops
that.

> **Note**
> At the time of writing the published constraints file is wrong — it pins
> three packages instead of the thirty-seven in the index, and `cffi` is not
> among them. The cause is fixed in `meta`, but the served file stays wrong
> until the next STABLE publish regenerates it. See
> [Known issue](#known-issue-the-published-constraints-file).
>
> Until then, `--only-binary=:all:` gets a **fresh** install through:
>
> ```sh
> export PIP_EXTRA_INDEX_URL="https://repo.zopen.community/pypi/wheels/simple/"
> pip install --only-binary=:all: snowflake-connector-python
> ```
>
> It makes pip skip any version it cannot find a compatible wheel for, so it
> falls back to the one the index has. It is a weaker tool than the constraints
> file and not a general substitute: `--only-binary` has nothing to say about a
> package that is *already installed*. Prefer the constraints file once it is
> serving the right content.

## Using it

```python
import snowflake.connector

conn = snowflake.connector.connect(
    account="<account_identifier>",
    user="<user>",
    password="<password>",
    warehouse="<warehouse>",
    database="<database>",
    schema="<schema>",
)

with conn.cursor() as cur:
    cur.execute("select current_version()")
    print(cur.fetchone()[0])

conn.close()
```

Key-pair authentication, which is usually the better fit for a batch job:

```python
from cryptography.hazmat.primitives import serialization

with open("rsa_key.p8", "rb") as fh:
    key = serialization.load_pem_private_key(fh.read(), password=None)

conn = snowflake.connector.connect(
    account="<account_identifier>",
    user="<user>",
    private_key=key.private_bytes(
        encoding=serialization.Encoding.DER,
        format=serialization.PrivateFormat.PKCS8,
        encryption_algorithm=serialization.NoEncryption(),
    ),
)
```

## Result format

This connector is built without its optional C++ result-set decoder, so it asks
Snowflake for the JSON result format instead of Arrow. The fallback is
automatic — there is nothing to configure — and the connector notes it at debug
level:

```
Cannot use arrow result format, fallback to json format
```

You will also see one warning on the first import, which is expected and not an
error:

```
Failed to import ArrowResult. No Apache Arrow result set format can be used.
ImportError: No module named 'snowflake.connector.nanoarrow_arrow_iterator'
```

It comes from a `logger.warning` in `cursor.py`, and since the connector
installs a `NullHandler` and most programs configure no handler at all, Python's
last-resort handler prints it to stderr. Silence it if it is noise:

```python
import logging
logging.getLogger("snowflake.connector.cursor").setLevel(logging.ERROR)
```

Everything returns the same values it would anywhere else. Large result sets
parse more slowly than they would with the Arrow decoder.

Two consequences worth knowing:

* `fetch_pandas_all()` and `fetch_pandas_batches()` are unavailable. They are
  Arrow-only and raise `NotSupportedError` rather than returning bad data.
  Use `fetchall()` and build the DataFrame yourself.
* `write_pandas()` is unaffected.

### Why Arrow is off

Not because it fails to build — because getting it *subtly* wrong is worse than
not having it.

Arrow's IPC format is little-endian by definition, and z/OS is big-endian, so
every buffer has to be byte-swapped on the way in. nanoarrow knows this;
`ArrowIpcDecoderNeedsSwapEndian()` compares the stream's endianness against the
system's, so the path exists and is not obviously wrong. It has simply never
been exercised on this platform, and a decoder that swaps *almost* correctly
returns wrong numbers rather than an error. In a database driver that is the
worst bug available, and the only thing it buys is faster parsing.

Upstream ships that decoder as a vendored nanoarrow plus flatcc and 24
converter sources behind a Cython module. It is genuinely optional: `setup.py`
skips it when `SNOWFLAKE_DISABLE_COMPILE_ARROW_EXTENSIONS` is set, and
`cursor.py` catches the `ImportError`, sets `CAN_USE_ARROW_RESULT_FORMAT` to
`False`, and switches the session to JSON.

Turning it on is a contained change for whoever wants to do the work: set the
variable to `false` in `buildenv`, add `cython>=3.1` to the build venv, and
prove the swap path with a round-trip against a real account — big integers,
decimals, timestamps and binary columns in particular. Until someone does that,
it stays off, and the port's check asserts the extension is absent so it cannot
come back by accident.

## Why this port exists

PyPI has no `py3-none-any` wheel for this package — only manylinux, macOS and
Windows binaries — so pip on z/OS falls through to the sdist. The sdist's
`[build-system]` requires names `cython>=3.1`; an isolated build installs it;
`_ABLE_TO_COMPILE_EXTENSIONS` comes out true; and pip tries to compile the C++
decoder. This port is what stops that happening by accident, and it publishes a
`py3-none-any` wheel so every interpreter shares one artifact.

## The minicore libraries

The connector vendors a small native library called *minicore*, shipped as one
directory per platform: `aix_ppc64`, `linux_{x86_64,aarch64}_{glibc,musl}`,
`macos_{x86_64,aarch64}` and `windows_x86_64`. None of them is z/OS.

`setup.py` normally strips the ones it does not need, but it gives up when it
cannot identify the platform it is on:

```python
keep = _minicore_native_subdir()
if keep is None:
    return
```

and `_minicore_native_subdir()` returns `None` for any machine that is not
x86_64, aarch64 or ppc64. On z/OS the "keep only the native one" path therefore
becomes "keep all nine", and the wheel comes out at **14 MB instead of 1.2 MB**
— most of it an AIX static archive that is 17 MB uncompressed.

This port removes those directories before building. Nothing needs them:
`snowflake/connector/__init__.py` already treats the library as optional.

```python
try:
    _core_loader.load()
except Exception:
    # This ensures the connector module loads even if the minicore library
    # is unavailable
    pass
```

The `minicore` package itself and its `__init__.py` are kept, because the
loader reaches it through `importlib.resources.files(...)` and an absent
package would turn a clean miss into an import error.

## What is checked

The port's check phase runs against each interpreter it builds for, without
needing a Snowflake account or any credentials in CI:

| check | what it would catch |
| --- | --- |
| version is the one this port builds | a stale wheel surviving in `dist/` |
| the Arrow C++ extension was not compiled in | a dropped environment variable silently re-enabling the unverified decoder |
| the connector falls back to the JSON result format | the extension absent but the fallback not engaging |
| the installed connector is pure Python | a compiled object in a tree three interpreters share — this is what caught the nine platforms of vendored minicore binaries |
| the DB-API 2.0 surface is present | a wheel that imports but is unusable |
| the type converter binds Python values | an encoding fault in the bind path |
| a 64-bit integer binds identically on big-endian | a byte-order mistake wider than 32 bits |
| native byte order is big-endian | the check running somewhere it does not mean anything |
| key-pair auth primitives work | a broken ported `cryptography`, or `jwt` |
| cryptography is new enough to be the ported one | the interpreter's own cryptography 3.3.2 winning over the ported one |
| pyOpenSSL imports against that cryptography | the version clash, at the point it actually breaks |
| the boto import chain resolves | the same clash one step out, via `urllib3.contrib.pyopenssl` |
| the CA bundle is present and readable | TLS failing only at connect time |
| optional dependencies degrade instead of raising | `boto3` absence turning into an import error |

## Dependencies

`cryptography` is the only runtime dependency that is not pure Python — it is a
Rust extension with a per-interpreter ABI — so it comes from its own zopen port
and is declared in `ZOPEN_RUNTIME_DEPS`.

Everything else the connector needs is pure Python and is bundled into
`lib/python` at install time, because none of it is a zopen port and one copy
can safely serve all three interpreters.

`boto3` and `botocore` are bundled because the connector's own metadata lists
them, not because importing it needs them: `options.py` routes both through
`MissingOptionalDependency`, so their absence surfaces only when you transfer
to an S3-backed stage. They are also most of the bundle's size — `botocore`
alone is about 16 MB. Dropping them is safe if that ever matters more than
completeness, and the failure it introduces is a clear one.

## A note on the interpreter's own cryptography

The IBM Open Enterprise SDK interpreters ship a copy of `cryptography` in their
own `site-packages`, and it is old — 3.3.2 at the time of writing, against the
46+ this connector requires. The ported `cryptography` is the one that should
win, and does.

It is worth knowing because of how it fails if it ever does not. You do not get
a version error; you get a `TypeError` from inside `cryptography.utils` while
importing pyOpenSSL:

```
File ".../OpenSSL/crypto.py", line 1788, in <module>
    utils.deprecated(
TypeError: deprecated() got an unexpected keyword argument 'name'
```

which names neither package nor version. The port's checks assert the version
directly so that this surfaces as what it is.

## Known issue: the published constraints file

The zopen Python documentation tells you to use `PIP_CONSTRAINT` alongside the
wheel index:

```sh
export PIP_EXTRA_INDEX_URL="https://repo.zopen.community/pypi/wheels/simple/"
export PIP_CONSTRAINT="https://repo.zopen.community/pulp/content/constraints/zopen-constraints.txt"
```

At the time of writing that file pins only `bcrypt`, `jellyfish` and `psutil`,
while the wheel index serves 37 packages. `cffi` and `cryptography` are both in
the index and neither is pinned, which is why the `pip install` instructions
above use `--only-binary=:all:` instead of relying on the constraints file.

The cause was that the constraints file is generated from whichever wheel index
the publish job ran against, but was published to a single fixed location shared
by both build lines — so a `DEV` publish, whose index carries three packages,
overwrote the `STABLE` one.

Fixed in `meta` (`c306a6ac`): STABLE keeps the documented path and any other
index gets its own, so `wheels-dev` now publishes to `constraints-dev`. The
served file stays wrong until the next STABLE publish regenerates it, which is
why the install instructions above carry a workaround.

## Building

```sh
zopen build -v
```

The build is pure Python and needs no compiler. `SNOWFLAKE_DISABLE_COMPILE_ARROW_EXTENSIONS`,
`ZOPEN_PYTHON_BUILD_ISOLATION=false` and `ZOPEN_PYTHON_BUILD_SKIP_DEP_CHECK=true`
are set in `buildenv` and are load-bearing; the note at the top of that file
explains what each one is holding up.
