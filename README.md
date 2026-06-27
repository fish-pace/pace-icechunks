# PACE OCI Level-3 Chlorophyll Icechunk Store

This Icechunk repository provides virtual access to NASA PACE OCI Level-3 chlorophyll (short_name `PACE_OCI_L3M_CHL`) products for **in-region AWS access**. The data remain in the original NASA S3 bucket and are accessed through virtual references, avoiding duplication while providing a Zarr-compatible interface.

The repository is organized by temporal resolution and spatial resolution from the cooresponding files in the `PACE_OCI_L3M_CHL` collection.

```text
daily/
  4km/
    chunks_512/
    chunks_16/
  0p1deg/
    chunks_512/
    chunks_16/

8day/
    ...

monthly/
    ...
```

Each temporal/spatial-resolution dataset is split into two groups based on the chunking of the original NASA NetCDF files:

- **`chunks_512`**: Files with chlorophyll chunked as `(time=1, lat=512, lon=1024)`.
- **`chunks_16`**: Files with chlorophyll chunked as `(time=1, lat=16, lon=1024)`.

The NASA archive transitioned from 512-row latitude chunks to 16-row latitude chunks during the record. Because Zarr currently requires arrays appended along a dimension to have identical chunk shapes, the two chunk layouts must be stored as separate datasets.

Each dataset includes a `time` coordinate derived from the start date in the filename, allowing it to be opened as a standard xarray dataset with `xr.open_zarr()`.

## Read in a dataset (requires you are in AWS us-west-2)

See `pace-icechunk-examples.ipynb` for more examples and plot examples. See `environment.yml` for the basic environment needed.

NASA Earthdata requires authentication. Use `earthaccess` to get the s3 credentials.

```
import earthaccess
import icechunk as ic
import xarray as xr

url = "https://data.source.coop/fish-pace/pace-oci/inregion/PACE_OCI_L3M_CHL"

storage = ic.http_storage(url)

auth = earthaccess.login()
s3_credentials = auth.get_s3_credentials(daac="OBDAAC")

url_prefix = "s3://ob-cumulus-prod-public/"

virtual_creds = ic.credentials.containers_credentials({
    url_prefix: ic.credentials.s3_credentials(
        access_key_id=s3_credentials["accessKeyId"],
        secret_access_key=s3_credentials["secretAccessKey"],
        session_token=s3_credentials["sessionToken"],
    )
})

repo = ic.Repository.open(
    storage,
    authorize_virtual_chunk_access=virtual_creds,
)

store = repo.readonly_session("main").store

ds512 = xr.open_zarr(
    store,
    consolidated=False,
    group="daily/0p1deg/chunks_512",
)

ds16 = xr.open_zarr(
    store,
    consolidated=False,
    group="daily/0p1deg/chunks_16",
)
```

We can concat the 2 groups to make on dataset with all times.

```
# Concat the groups
ds = xr.concat(
    [ds512, ds16],
    dim="time",
    coords="minimal",
    compat="override",
    combine_attrs="override",
).sortby("time")
```



