# PACE OCI Level-3 Chlorophyll Icechunk Store

This Icechunk repository provides virtual access to NASA PACE OCI Level-3  products for **in-region AWS access**. The data remain in the original NASA S3 bucket and are accessed through virtual references, avoiding duplication while providing a Zarr-compatible interface.

There are the following Icechunk stores:
* `PACE_OCI_L3M_CHL`
* `PACE_OCI_L3M_RRS`

Each L3 product has daily, monthly, and 8D at 2 different grids 0p1deg and 4km. The group names are: 
* `daily/0p1deg` `daily/4km`
* `monthly/0p1deg` `monthly/4km`
* `8Day/0p1deg` `8Day/4km`

The `PACE_OCI_L3M_CHL` daily groups are separated into subgroups based on differences in chunking before 2026-02 and afterwards.
* `daily/0p1deg/chunks_512` +  `daily/0p1deg/chunks_16` and `daily/4km/chunks_512` +  `daily/4km/chunks_16`

- **`chunks_512`**: Files with chlorophyll chunked as `(time=1, lat=512, lon=1024)`.
- **`chunks_16`**: Files with chlorophyll chunked as `(time=1, lat=16, lon=1024)`.

Each dataset includes a `time` coordinate derived from the start date in the filename, allowing it to be opened as a standard xarray dataset with `xr.open_zarr()`.

## Read in a dataset (requires you are in AWS us-west-2)

See `pace-icechunk-examples.ipynb` for more examples and plot examples. See `environment.yml` for the basic environment needed. NASA Earthdata requires authentication. Use `earthaccess` to get the s3 credentials.

```
ds = create_ds("PACE_OCI_L3M_RRS", "daily/0p1deg")
ds
```
Using the function below:
```
import earthaccess
import icechunk as ic
import xarray as xr

def create_ds(
    product="PACE_OCI_L3M_RRS",
    group="daily/0p1deg",
):
    url = f"https://data.source.coop/fish-pace/pace-oci/inregion/{product}"
    
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
    
    store = repo = ic.Repository.open(
        storage,
        authorize_virtual_chunk_access=virtual_creds,
    ).readonly_session("main").store

    ds = xr.open_zarr(
        store,
        consolidated=False,
        group=group,
    )

    return ds
```




