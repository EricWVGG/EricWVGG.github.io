# Building a Store Finder with NextJS, Google Maps and Sanity

I’ve been asked to install “store locator” widgets a few times over the years. I was surprised to learn that it’s even a product that people pay for…

This tutorial assumes you have basic experience in NextJS, Sanity, and Typescript, and that Sanity is set up for automatic type generation. (If you aren't using that last one, you can replace the Sanity.Whatever types with `any` or manually written types… but boy you are missing out.)

I’ll assume you know little or nothing about Google Maps, though.

// this is a first-draft haven’t tested this code yet

## Dependencies

Yup React developers love us some dependency hell, let's go…

```
pn install google-map-react usehooks-ts
```

## Add a map location schema to Sanity

You’ll need to create a basic schema in Sanity to store the map locations.

```typescript
import { defineField, defineType } from "sanity"

// This is our new document type…
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
      validation: (rule) => rule.min(-90).max(90),
      fieldset: "coordinates",
    }),
    defineField({
      name: "longitude",
      type: "number",
      validation: (rule) => rule.min(-180).max(180),
      fieldset: "coordinates",
    }),
  ],
  // create a “coordinates” field set to keep things tidy
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

(Yes, -90 to 90 and -180 to 180 are the valid ranges for latitude and longitude figures!)

Add this to your Sanity schema…

```typescript
import { mapLocationSchema } from "./mapLocationSchema"

export const schema: { types: SchemaTypeDefinition[] } = {
  types: [
    // ... the rest of your schema,
    mapLocationSchema,
  ],
}
```

And create a Groq query for retrieval.

```typescript
import { defineQuery } from "next-sanity"

export const mapLocationsQuery = defineQuery(`
  *[_type == 'mapLocation'] {
    _id
  }
`)
```

## retrieve the data in NextJS

### .env and constants

Todo: instructions on creating a key

In your `.env` file, add your Google Maps API key:

```
NEXT_PUBLIC_GOOGLE_MAPS_KEY=some-string-from-google
```

You’ll want some default values for the map. If you keep a global constants file somewhere, add these here; otherwise they can just go in place of the `import {...} from @/const` declaration in the `Map` component below.

```
export const MAP_DEFAULT_LAT = 41.8277584
export const MAP_DEFAULT_LNG = -87.6620778
export const MAP_DEFAULT_ZOOM = 10
export const MAP_CLUSTER_RADIUS = 120
export const MAP_MAX_ZOOM = 20
export const GOOGLE_MAPS_KEY = process.env.NEXT_PUBLIC_GOOGLE_MAPS_KEY
```

### NextJS Page

Create a new route in NextJS (I’m assuming you’re using the App Router here).

1. create a folder in `/app` called `/locations`.
2. create a file in `/locations` called `page.tsx`
3. draft the file thusly:

```
import { sanityFetch } from "@/sanity/lib/live" // < this is wherever you load your Sanity client from
import { mapLocationsQuery } from "@/sanity/queries" // < this is wherever you keep your Sanity queries
import { Map } from "@/ui" // < this is wherever you export React components from

export default async function MapLocationsPage() {
  const { data } = await sanityFetch({
    query: mapLocationsQuery,
  })
  if (!data) throw new Error("error loading map locations from Sanity")
  return (
    <Map locations={data} />
  )
}
```

### React Map component

We're going to do this in three steps…
1. Make a map that shows pins.
2. Make a popup for pin details.
3. Add "clusters" for closely set pins.

I’ve added "insertion points" for the latter steps below; just ignore them for now.

Create a new component called `Map.tsx`:
```typescript
"use client"

import { useState, useMemo, useRef } from "react"
import { useWindowSize } from "usehooks-ts"
import { MAP_DEFAULT_LAT, MAP_DEFAULT_LNG, MAP_DEFAULT_ZOOM, GOOGLE_MAPS_KEY, MAP_CLUSTER_RADIUS, MAP_MAX_ZOOM } from "@/const" // see above
import { Pin } from './Pin'
// insert: popup imports
// insert: cluster imports

export const Map = ({ locations }: { locations: Sanity.MapLocationsQueryResult }) => {
  /*
    Google maps requires that the width and height of the map is _explicitly set in the style attribute_.
    It doesn't have to be in pixels — you'll see that I'm using dvw and dvh units below — but it does have to be in "style".
    Putting it in a className will not work!
  */
  const { width } = useWindowSize()
  
  // insert: popup states
  
  // insert: cluster states
  
  // insert: cluster points memo
  
  // insert: cluster hook
  
  // insert: cluster click action

  return (
    <main style={{ height: width < 744 ? "80dvh" : "63dvw", maxHeight: "80dvh", width: "100%" }}>
      {/* that’s the style attribute I warned you about ^ */}
      <GoogleMapReact
        bootstrapURLKeys={{ key: googleMapsKey }}
        defaultZoom={defaultZoom}
        options={{
          clickableIcons: false,
        }}
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
        
        {/* insert: cluster points */}
        
        {/* insert: popup component */}
      </GoogleMapReact>
      
      {/* Google Maps bug fix, see below */}
      <style type="text/css" dangerouslySetInnerHTML={{__html: `
        .gm-style div > img {
          position: absolute;
        }
      `}}>
    </main>
  )
}

// insert: cluster marker type
```

There’s two fixes here for unexpected behaviors. By default, Google makes certain locations like public parks clickable. That can lead to unexpected results if you have a map pin near one of these locations. `clickableIcons: false` fixes that.

That last style tag is an important bug fix. [Embedded maps map have a missing row of map tiles on the bottom row.](https://stackoverflow.com/questions/41544151/google-maps-missing-row-of-tiles-on-chrome) It's been an issue for years. You can put this fix in a style tag as per above, or move the style into your global stylesheet (recommended).

We’ll also need a component for our Pin. Create the following file `Pin.tsx`:

```typescript
interface PinProps {
  lat: number
  lng: number;
  onClick?: any // yes I'm sometimes lazy about onClick types
}

export const Pin = ({ lat, lng, onClick }: PinProps) => (
  <div 
    style={{
      position: 'relative',
      display: 'flex',
      justifyContent: 'center',
      alignItems: 'center',
      width: '30px',
      height: '30px',
      transform: 'translateX(-50%) translateY(-50%)',
    }} 
    lat={lat} 
    lng={lng} 
    onClick={onClick} 
  />
)
```

This is the basics, you should have a working map now.

### Make the map load dynamically

Since this map has a pretty heavy payload of dependencies, we're going to use `next/dynamic` to dynamically load the necessary javascript upon page load.

First, add this import up top:
```typescript
import dynamic from "next/dynamic"
```

Replace this line:
```
// old: export const Map = ({ locations }: { locations: Sanity.MapLocationsQueryResult }) => {
const UnhydratedMap = ({ locations }: { locations: Sanity.MapLocationsQueryResult }) => {
```

And add this at the bottom of the document:
```
export const Map = dynamic(() => Promise.resolve(UnhydratedMap), { ssr: false })
```

Easy! You can stop now if that’s all you want out of this project, but popups and clusters are usually desirable…

### Popups

Let's make a popup that shows a name and address when the user clicks on a pin.

When I built this, my first impulse was to put a popup inside each pin with it's own show/hide state. That approach doesn’t scale well, it’s not uncommon to have hundreds of pins (more on that when we get to clusters).

So instead, we're going to make one `<Popup />` component, with a dynamic position. Create `Popup.tsx`:

```typescript
interface PopupProps {
  lat?: number;
  lng?: number;
  location?: Member<Sanity.MapLocationsQueryResult> 
}

export const Popup = ({ lat, lng, location }: PopupProps) => (
  <div
    lat={lat}
    lng={lng}
    style={{
      display: (!!lat && !!lng) ? 'block' : 'none',
      background: 'black',
      foreground: 'white'
    }}
  >
    <div><strong>{location?.name}</strong></div>
    <div><em>{location?.streetAddress}</em></div>
  </div>
)
```

Now let’s add the Popup and functionality to the Map component.

replace `// insert: popup imports` with:
```typescript
import { Popup } from './Popup'
```

replace `// insert: popup states` with:
```typescript
const [activeLocation, setActiveLocation] = useState<Member<Sanity.MapLocationsQueryResult> | null>(null)
```

replace `{/* insert: popup component */}` with:
```typescript
<Popup 
  location={activeLocation || undefined} 
  lat={activeLocation?.geoLocation?.latitude} 
  lng={activeLocation?.geoLocation?.longitude} 
/>
```

replace `/* insert: popup pin clicks */` with:
```typescript
onClick={() => setActiveLocation(mapItem.location)} 
```

Recap: we're now tracking an “active location”. Upon clicking a map pin, that location is set.

The popup dynamically keeps `activeLocation`, and hides itself if no lat/lng is selected.

### Clusters

When multiple pins are close to each other on a map, they become indiscernible and unclickable. If you're dealing with even more than a handful of pins, this becomes necessary fast.

Add these dependencies:
```
pn install supercluster use-supercluster
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

Instead of simply mapping over the locations, we instead need to map over structured points that can be read by SuperCluster. Since the calculation isn’t quite cheap, we’ll use `useMemo()` to emsmarten it.

Replace `// insert: cluster marker type` with:
```typescript
interface MarkerProperties {
  cluster: boolean
  cluster_id?: number
  point_count?: number
  point_count_abbreviated?: number
  locationData: Member<Sanity.MapLocationsQueryResult>
}
```

Replace `// insert: cluster points memo` with:
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
            coordinates: [location.geoLocation!.longitude, location.geoLocation!.latitude],
          },
        }) as PointFeature<MarkerProperties>
    ),
  [locations]
)
```

Now we’ll set up our “clusterizer”. Replace `// insert: cluster hook` with:
```typescript
const { clusters, supercluster } = useSupercluster({
  points,
  bounds,
  zoom,
  options: { radius: CLUSTER_RADIUS, maxZoom: MAX_ZOOM },
})
```

We’ll need a click action. Replace `// insert: cluster click action` with:

```typescript
const zoomOnClusterAction = (clusterId: number | string, lat: number, lng: number) => {
  if (typeof supercluster === "undefined") return
  const newZoom = Math.min(
    supercluster.getClusterExpansionZoom(
      typeof clusterId === "number" 
        ? clusterId 
        : Number(clusterId)
    ), 
    MAX_MAP_ZOOM
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

We need to replace the location pins with a mix of clusters and pins. Replace this:
```typescript
{locations.map(({latitude, longitude}, i) => (
  // etc.
))}
```
… with this:
```typescript
{clusters.map((mapItem, i) => {
  const [longitude, latitude] = mapItem.geometry.coordinates
  const { cluster: isCluster, point_count: pointCount } = mapItem.properties
  return isCluster && pointCount ? (
    <Cluster 
      key={`cluster-${mapItem.id}`} 
      lat={latitude}
      lng={longitude} 
      pointCount={pointCount} 
      totalPoints={points.length}
      onClick={() => zoomOnCluster(mapItem.id!, latitude, longitude)} 
    />
  ) : (
    <Pin 
      key={`marker-${i}`} 
      lat={latitude} 
      lng={longitude} 
      onClick={() => setActiveLocation(mapItem.location)}
    />
  )
})}
```

I hope that's pretty straightforward: `useSuperCluster` generates a new mix of Cluster and Pin items; output the correct components accordingly.

Finally, we’ll need a `<Cluster />` component, very similar to the `<Pin />` component. Create `Cluster.tsx`:

```typescript
interface ClusterProps {
  lat: number;
  lng: number;
  pointCount: number;
  totalPoints: number;
  onClick: any
}

export const Cluster = ({ lat, lng, pointCount, totalPoints, ...props }: ClusterProps) => (
  <div
    {...props}
    style={{
      width: `${40 + (pointCount / totalPoints) * 10}px`,
      height: `${40 + (pointCount / totalPoints) * 10}px`,
      position: 'relative',
      display: 'flex',
      justifyContent: 'center',
      alignItems: 'center',
      transform: 'translateX(-50%) translateY(-50%)',
      borderRadius: '9999px',
      background: 'blue',
      color: 'white',
      fontSize: '12px',
    }}
  >
    x{pointCount}
  </div>
)
```

