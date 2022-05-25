# Digital Agenda Data Visualization Tool - Backend

This is the backend for the Digital Agenda **Data Visualisation Tool**.

## Docker Setup

Build the images:

```shell
docker/build.sh
```

Bring up the app and database:

```shell
COMPOSE_FILE=docker/docker-compose.yml
docker-compose up -d
```

Drop into an app container shell, e.g.:

```shell
docker-compose exec app-http bash
```

to perform Django initialization routines:

```shell
django-admin migrate
django-admin collectstatic
django-admin createsuperuser
```

Note that `createsuperuser` can rely on envars `DJANGO_SUPERUSER_EMAIL` & `DJANGO_SUPERUSER_PASSWORD`, if available:
```shell
django-admin createsuperuser --noinput
```

## Initial metadata

The `core` app (`digital_agenda.apps.core`) provides fixtures for seeding:
- data sources
- indicator groups
- indicators
- breakdown groups
- breakdowns

To import these fixtures (order matters):

```shell
django-admin loaddata datasources
django-admin loaddata indicatorgroups
django-admin loaddata indicators
django-admin loaddata indicatorgrouplinks
django-admin loaddata breakdowngroups
django-admin loaddata breakdowns
django-admin loaddata breakdowngrouplinks
```

Alternative fixtures can be produced from Excel exports of code lists from the previous version:
```shell
python scripts/mk_data_sources_fixture.py source.xls
python scripts/mk_indicator_groups_fixture.py indicator-group.xls
python scripts/mk_indicators_fixture.py indicator.xls --groups-excel-file=indicator-group.xls --data-sources-excel-file=source.xls
python scripts/mk_breakdown_groups_fixture.py breakdown-group.xls 
python scripts/mk_breakdowns_fixture.py breakdown.xls --groups-excel-file=breakdown-group.xls
```
The scripts for the indicators and breakdowns fixtures also produce the "links" fixtures with their respective groups.
For indicators, integrity checks can be performed against the indicator groups and data sources Excel files. 
For breakdowns, validation of the group can be done versus the breakdown groups Excel file.  
In both cases, warnings are issued for unknown references, and those association records are **not** included in the links fixture.

## Eurostat data import

### Bulk Metadata

Import Eurostat bulk metadata for each source dataset, e.g.:

```shell
django-admin estat-import-bulk-meta isoc_bde15dec \
--indicator=indic_is \
--breakdown=sizen_r2 \
--unit=unit \
--country=geo \
--period=time \
--delete-existing
```

Eurostat dataset dimension values are imported in a disabled state, 
to avoid accidentally loading large amounts of unwanted facts.
Enable the appropriate values using the command:

```shell
django-admin estat-enable-dim-values <JSON file>
```

where the input file has a structure mapping dataset, dimension and values, e.g.:

```json
{
  "isoc_bde15dec": {
    "indic_is": [
      "e_crman",
      "e_ebuy"
    ]
  },
  "isoc_bde15ee": {
    "indic_is": [
      "e_igov",
      "e_igov2",
      "e_igov3",
      "e_igovpr",
      "e_igov2pr",
      "e_igovrt",
      "e_igov2rt"
    ]
  }
```

See the `examples` directory for sample input file.
Note that some indicators are present in multiple datasets. Avoid enabling an indicator 
value multiple times - while this will not be a problem during the bulk data import, it
can cause data integrity problems during the final data processing (see the dashboard 
queries section below).

### Bulk data

Once the metadata is in place, load the bulk metadata for each dataset, e.g.:

```shell
django-admin estat-import-bulk-data isoc_bde15dec
```

### Dashboard Queries

Imported bulk data is processed into the final reporting format using SQL queries defined 
in the Dashboard.

#### Database preparation

Create a read-only user for dashboard usage, matching the values 
in the settings items `POSTGRES_DASHBOARD_USER` & `POSTGRES_DASHBOARD_PASSWORD`:

```postgresql
CREATE ROLE digital_agenda_ro WITH
    LOGIN
    NOSUPERUSER
    INHERIT
    NOCREATEDB
    NOCREATEROLE
    NOREPLICATION
    PASSWORD 'digital_agenda_ro';
```

Allow the read-only user to read the bulk tables:

```postgresql
GRANT SELECT ON TABLE
    public.estat_datasets,
    public.estat_dimensions,
    public.estat_dim_values,
    public.estat_facts
TO dagr_ro;
```

Note: non-superuser application users must belong to the `Dashboard` group 
(pre-created during migrations) to be able to view and edit dashboards and queries.
The dashboard app is available at the `/dashboard/` URL.

Add dashboards and SQL queries for importing bulk facts into the core tables - an example 
query is available in `examples/ENT2_after_2009.sql`. Note that while each dashboard supports 
multiple queries, only the first is used during the final import stage, by specifying the 
dashboard's slug as the second parameter:

```shell
django-admin import-with-query estat ent2_after_2009
```
The first parameter above refers to the data source - one for ESTAT was created during migrations.

Be aware that core facts must have unique indicator/breakdown/unit/country/period combinations. 
Consequently, dashboard queries must either enforce this, or avoid data duplication, e.g. from 
indicators that are present in multiple datasets. Otherwise, the `import-with-query` command will 
terminate with data integrity errors, and the import must be re-run after fixing the query.
