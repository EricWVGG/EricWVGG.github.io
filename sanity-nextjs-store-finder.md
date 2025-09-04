# Building a Store Finder with NextJS, Google Maps and Sanity

I’ve been asked to install “store locator” widgets a few times over the years. I was surprised to learn that it’s even a product that people pay for…

This tutorial assumes you have basic experience in NextJS, Sanity, and Typescript, and that Sanity is set up for automatic type generation. (If you aren't using that last one, you can replace the Sanity.Whatever types with `any` or manually written types… but boy you are missing out.) And if you’re not using NextJS, these instructions should be pretty easy to adapt.

You don’t need to know much about Google Maps, but [you will need an API key](https://developers.google.com/maps/documentation/javascript/get-api-key).

// this is a first-draft, haven’t tested this code yet

## Dependencies and .env

We all live in dependency hell, let's go…

```
npm install google-map-react usehooks-ts
```

In your .env file, add your Google Maps API key:

```
NEXT_PUBLIC_GOOGLE_MAPS_KEY=some-string-from-google
```

## Add a map location schema to Sanity

You’ll need to create a basic schema in Sanity to store the map locations.

```typescript
import { defineField, defineType } from "sanity"

export const mapLocationSchema = defineType({
  name: "mapLocation",
  title: "Stockist",
  type: "document",
  orderings: [
    {
      title: "Location Name",
      name: "locationName",
      by: [{ field: "name", direction: "asc" }],
    },
  ],
  fields: [
    defineField({
      name: "name",
      type: "string",
      validation: (Rule) => Rule.required().max(60),
    }),
    defineField({
      name: "streetAddress",
      type: "string",
    }),
    defineField({
      name: "city",
      type: "string",
      validation: (rule) => rule.required(),
    }),
    defineField({
      name: "state",
      type: "string",
      validation: (rule) => rule.required().min(2).max(2),
    }),
    defineField({
      name: "postalCode",
      type: "string",
      validation: (rule) => rule.min(5).max(5),
    }),
    defineField({
      name: "latitude",
      type: "number",
      validation: (rule) => rule.required().min(-90).max(90),
      fieldset: "coordinates",
    }),
    defineField({
      name: "longitude",
      type: "number",
      validation: (rule) => rule.required().min(-180).max(180),
      fieldset: "coordinates",
    }),
  ],
  // a “coordinates” grouping to keep things tidy
  fieldsets: [
    {
      title: "Coordinates",
      name: "coordinates",
      options: {
        columns: 2,
      },
    },
  ],
})
```

Yes, -90 to 90 and -180 to 180 are the valid ranges for latitude and longitude figures!

If you don’t have lat/lng coordinates, I’m going to follow up this tutorial with [two approaches for getting this data](#follow-up-tutorial); for now, just fake it with some made-up numbers.

Add this to your existing Sanity schema:

```typescript
import { mapLocationSchema } from "./mapLocationSchema"

export const schema: { types: SchemaTypeDefinition[] } = {
  types: [
    // … your existing schema,
    mapLocationSchema,
  ],
}
```

And create a Groq query for retrieval: `mapLocationsQuery.ts`

```typescript
import { defineQuery } from "next-sanity"

export const mapLocationsQuery = defineQuery(`
  *[_type == 'mapLocation']
`)
```

## NextJS Page

I’m assuming you’re using the App Router here, and use some kind of alias imports. If you’re not using alias imports, replace `@/` below with hard-coded paths to your Sanity client, the query you wrote, and the `Map.tsx` file you'll be making next.

Create `/app/locations/page.tsx`:

```typescript
import { sanityFetch, mapLocationsQuery } from "@/sanity"
import { Map } from "@/ui"

export default async function MapLocationsPage() {
  const { data } = await sanityFetch({
    query: mapLocationsQuery,
  })
  if (!data) throw new Error("error loading map locations from Sanity")
  return <Map locations={data} />
}
```

## React components

We're going to do this in three steps…

1. Make a map that shows pins.
2. Make a popup for pin details.
3. Add "clusters" for closely set pins.

If all you want is a map with pins, steps 2 and 3 are totally optional!

I’ve added "insertion points" for the latter steps; just ignore them for now.

### Basic map with pins

#### Map component

Create a new component called `Map.tsx`:

```typescript
'use client'

import { useState, useMemo, useRef } from 'react'
import { useWindowSize } from 'usehooks-ts'
import { Pin } from './Pin'
// insert: popup imports
// insert: cluster imports

export const MAP_DEFAULT_LAT = 41.8277584
export const MAP_DEFAULT_LNG = -87.6620778
export const MAP_DEFAULT_ZOOM = 10
export const MAP_CLUSTER_RADIUS = 120
export const MAP_MAX_ZOOM = 20
export const GOOGLE_MAPS_KEY = process.env.NEXT_PUBLIC_GOOGLE_MAPS_KEY

const UnhydratedMap = ({ locations }: { locations: Sanity.MapLocationsQueryResult }) => {
  const { width } = useWindowSize()

  // insert: popup states

  // insert: cluster states

  // insert: cluster points

  // insert: clusterizer

  // insert: cluster click action

  return (
    <main
      // see NOTE 1
      style={&123;
        height: width < 744 ? "80dvh" : "63dvw",
        width: "100%"
      &125;}
    >
      <GoogleMapReact
        bootstrapURLKeys={{ key: googleMapsKey }}
        defaultZoom={defaultZoom}
        options={&123;
          // see NOTE 2
          clickableIcons: false,
        &125;}
        /* insert: cluster map capabilities */
      >
        {locations.map(({latitude, longitude}, i) => (
          <Pin
            key={`marker-${i}`}
            lat={latitude}
            lng={longitude}
            /* insert: popup pin clicks */
          />
        ))}

        {/* insert: cluster components */}

        {/* insert: popup component */}
      </GoogleMapReact>

      {/* see NOTE 3 */}
      <style type="text/css">
        {`.gm-style div > img {position: absolute;}`}
      </style>
    </main>
  )
}

// insert: cluster marker type

export const Map = dynamic(() => Promise.resolve(UnhydratedMap), { ssr: false })
```

#### Pin component

We’ll also need a component for our Pin. Create the following file `Pin.tsx`:

```typescript
'use client'

interface PinProps {
  lat: number
  lng: number;
  onClickAction?: any // yes I'm sometimes lazy about onClick types… they’re hard!
}

export const Pin = ({ lat, lng, onClickAction }: PinProps) => (
  <div
    style={&123;
      position: 'relative',
      display: 'flex',
      justifyContent: 'center',
      alignItems: 'center',
      width: '30px',
      height: '30px',
      transform: 'translateX(-50%) translateY(-50%)',
    &125;}
    lat={lat}
    lng={lng}
    onClick={onClickAction}
  />
)
```

#### Important fixes here for unexpected behaviors

<strong>NOTE 1</strong>: Google maps requires that the width and height of the map is _explicitly set in the style attribute_. It doesn't have to be in pixels — you'll see that I'm using dvw and dvh units below — but it does have to be in a `style=` attribute. <em>Putting it in a `className` will not work!</em>

<strong>NOTE 2</strong>: By default, Google makes certain locations like public parks clickable. That can lead to unexpected results if you have a map pin near one of these locations. `clickableIcons: false` fixes that.

<strong>NOTE 3</strong>: [Embedded maps may have a missing row of map tiles on the bottom row.](https://stackoverflow.com/questions/41544151/google-maps-missing-row-of-tiles-on-chrome) It's been an issue for years. That style declaration fixes it.

#### What’s this about next/dynamic?

Google Maps introduces a lot of dependencies into your project. By using `next/dynamic`, this stuff is kept outside your usual site javascript payload, and only loaded on demand.

Technically it’s not necessary; you may need to remove that if you’re using a completely static NextJS build.

### Popups

Let's make a popup that shows a name and address when the user clicks on a pin.

#### Popup Component

When I built this, my first impulse was to put a popup inside each pin with it's own show/hide state. That approach doesn’t scale well; it’s not uncommon to have hundreds of pins (more on that when we get to clusters).

So instead, we're going to make one `<Popup />` component, with a dynamic position. Create `Popup.tsx`:

```typescript
'use client'

interface PopupProps {
  lat?: number;
  lng?: number;
  location?: Member<Sanity.MapLocationsQueryResult>
}

export const Popup = ({ lat, lng, location }: PopupProps) => (
  <div
    lat={lat}
    lng={lng}
    style={&123;
      display: (!!lat && !!lng) ? 'block' : 'none',
      background: 'black',
      foreground: 'white'
    &125;}
  >
    <div><strong>{location?.name}</strong></div>
    <div><em>{location?.streetAddress}</em></div>
  </div>
)
```

#### Add the Popup and functionality to the Map component.

replace `// insert: popup imports` with:

```typescript
import { Popup } from "./Popup"
```

replace `// insert: popup states` with:

```typescript
const [activeLocation, setActiveLocation] = useState<Member<Sanity.MapLocationsQueryResult> | null>(null)
```

replace `{/* insert: popup component */}` with:

```typescript
<Popup location={activeLocation || undefined} lat={activeLocation?.geoLocation?.latitude} lng={activeLocation?.geoLocation?.longitude} />
```

replace `/* insert: popup pin clicks */` with:

```typescript
onClick={() => setActiveLocation(mapItem.location)}
```

Recap: we're now tracking an “active location”. Upon clicking a map pin, that location is set.

The popup dynamically keeps `activeLocation`, and hides itself if no lat/lng is selected.

### Clusters

When multiple pins are close to each other on a map, they become indiscernible and unclickable. If you're dealing with even more than a handful of pins, this becomes necessary fast.

#### Cluster component

We’ll need a `<Cluster />` component. It’s very similar to the `<Pin />` component. Create `Cluster.tsx`:

```typescript
'use client'

interface ClusterProps {
  lat: number;
  lng: number;
  pointCount: number;
  totalPoints: number;
  onClickAction: any
}

export const Cluster = ({ lat, lng, pointCount, totalPoints, onClickAction }: ClusterProps) => {
  const length = 40 + (pointCount / totalPoints) * 10
  return (
    <div
      lat={lat}
      lng={lng}
      onClick={onClickAction}
      style={&123;
        width: length.toString() + 'px',
        height: length.toString() + 'px',
        position: 'relative',
        display: 'flex',
        justifyContent: 'center',
        alignItems: 'center',
        transform: 'translateX(-50%) translateY(-50%)',
        borderRadius: '9999px',
        background: 'blue',
        color: 'white',
        fontSize: '12px',
      &125;}
    >
      x{pointCount}
    </div>
  )
}
```

#### Add clusters to `<Map />` component

There’s a lot of steps here, so I’ve done my best to make them very granular. First, add these dependencies to the project:

```
npm install supercluster use-supercluster
```

Replace `// insert: cluster imports` with:

```typescript
import useSupercluster from "use-supercluster"
import { type PointFeature } from "supercluster"
import { Cluster } from "./Cluster"
```

Replace `// insert: cluster states` with:

```typescript
const [bounds, setBounds] = useState<[number, number, number, number]>([0, 0, 0, 0])
const [zoom, setZoom] = useState(defaultZoom)
const mapRef = useRef<any>(null)
```

Instead of simply mapping over the locations, we need to map over structured points that can be read by SuperCluster. To avoid rebuilding this array on every component render, we’ll use `useMemo()` to emsmarten things.

Replace `// insert: cluster points` with:

```typescript
const points = useMemo(
  () =>
    locations.map(
      (location) =>
        ({
          type: "Feature",
          properties: { cluster: false, locationData: location },
          geometry: {
            type: "Point",
            coordinates: [
              location.geoLocation!.longitude,
              location.geoLocation!.latitude
            ],
          },
        } as PointFeature<MarkerProperties>)
    ),
  [locations]
)
```

And replace `// insert: cluster marker type` with:

```typescript
interface MarkerProperties {
  cluster: boolean
  cluster_id?: number
  point_count?: number
  point_count_abbreviated?: number
  locationData: Member<Sanity.MapLocationsQueryResult>
}
```

Now we’ll set up a “clusterizer” (yes, it’s really called that) that turns our map points into a mix of clusters and pins. Replace `// insert: clusterizer` with:

```typescript
const { clusters, supercluster } = useSupercluster({
  points,
  bounds,
  zoom,
  options: {
    radius: CLUSTER_RADIUS,
    maxZoom: MAX_ZOOM
  },
})
```

We’ll need a click action for the cluster markers. Replace `// insert: cluster click action` with:

```typescript
const zoomOnClusterAction = (clusterId: number | string, lat: number, lng: number) => {
  if (typeof supercluster === "undefined") return
  const newZoom = Math.min(
    MAX_MAP_ZOOM,
    supercluster.getClusterExpansionZoom(typeof clusterId === "number" 
      ? clusterId 
      : Number(clusterId))
  )
  mapRef.current?.setZoom(newZoom)
  mapRef.current?.panTo({ lat, lng })
}
```

We need to give the Map the ability to update the zoom and bounds from our code. Replace `/* insert: cluster map capabilities */` with:

```typescript
yesIWantToUseGoogleMapApiInternals
onGoogleApiLoaded={({ map }) => {
  mapRef.current = map
}}
onChange={({ zoom, bounds }) => {
  setZoom(zoom)
  setBounds([bounds.nw.lng, bounds.se.lat, bounds.se.lng, bounds.nw.lat])
}}
```

Finally, we need to replace the location pins with a mix of clusters and pins. Delete this…

```typescript
{locations.map(({latitude, longitude}, i) => (
  // etc.
))}
```

… and replace `{/* insert: cluster components */}` with:

```typescript
{
  clusters.map((mapItem, i) => {
    const [longitude, latitude] = mapItem.geometry.coordinates
    const { cluster: isCluster, point_count: pointCount } = mapItem.properties
    return isCluster && pointCount ? (
      <Cluster
        key={`cluster-${mapItem.id}`}
        lat={latitude}
        lng={longitude}
        pointCount={pointCount}
        totalPoints={points.length}
        onClickAction={() => zoomOnCluster(mapItem.id!, latitude, longitude)}
      />
    ) : (
      <Pin
        key={`marker-${i}`}
        lat={latitude}
        lng={longitude}
        onClickAction={() => setActiveLocation(mapItem.location)}
      />
    )
  })
}
```

I hope that's pretty straightforward: `useSuperCluster` generates a new mix of Cluster and Pin items, and outputs the correct components accordingly.

Hey cool we’re done. Give it a try.

## Follow-up tutorial

I’m going to follow up this tutorial with two more; both regard different approaches to getting the latitude and longitude coordinates for locations from Google Maps.

- Coming soon: building a custom Sanity GetCoordinates input component!
- Coming soon: syncing a Google Spreadsheet into Sanity, with automatic Google Maps geolocation lookups. 