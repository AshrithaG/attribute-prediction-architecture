# 9. Operations

## The container

A CPU only image on a slim Python 3.12 base.

- The CPU build of torch is installed first, so pip does not pull CUDA wheels
  into an image that will never see a GPU.
- Configuration is copied in, then the sentence encoder is downloaded at the
  pinned revision during the build and cached inside the image.
- The image then sets model hub access to offline, so the running container
  never reaches out for a model and cannot silently pick up a different one.
- Source and scripts are copied last, so a code change does not invalidate the
  layer holding the model.
- A health check polls the health endpoint, with a startup grace period long
  enough for artifacts to load.

Model artifacts and catalog files are mounted at runtime rather than baked in.
They are gigabytes and they version on a different cadence than the code, so a
new model version needs no new image.

## Configuration

Environment variables cover the API token, the artifact run directory and the
database path. Everything else lives in version controlled YAML: encoder
identity and revision, index parameters, thresholds and routing bands,
calibration settings, and unit aliases.

Values that belong to the agreed contract are marked as frozen in the files
themselves, next to the values that are free to tune. Writing that distinction
into the config rather than into a document means the person changing a number
sees it at the moment they change it.

## Resource profile

- Roughly 3.5 to 4 GB of memory resident, dominated by the index and the
  cluster statistics.
- A few milliseconds per document for retrieval and scoring on CPU.
- Startup is dominated by loading artifacts, tens of seconds.
- A full artifact rebuild is about an hour and a half of CPU, almost entirely
  encoding the catalog.

## Failure handling

- **Wrong or missing token:** 401, permanent, no retry upstream.
- **Malformed record:** 422, permanent.
- **Encoder mismatch against the artifacts:** refuses to start. This is a fatal
  configuration error, and serving would produce confident nonsense.
- **Persistence unavailable:** the store endpoints report unavailable rather
  than pretending to be empty.
- **Duplicate delivery:** returns the stored response.

## Runbook notes

- The queue depth gauge and the routing counters are the two signals to alarm
  on. A queue that only grows means reviewers are behind. A sudden shift in the
  routing mix means either the input changed or the model did.
- The drift gauge compares live confidence against the evaluation baseline. It
  is the earliest indication that production text has moved away from catalog
  text.
- The configuration fingerprint on the health endpoint and on every stored
  prediction is how to answer "which thresholds produced this row".
- Rolling back a model version is a pointer move, and the promotion log says
  what was active when.
