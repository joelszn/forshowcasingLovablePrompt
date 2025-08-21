Project: MyRiskMap — Website MVP
Audience: General public (U.S. residents). Solo-maintained, low-cost, "build once" site.

## Objectives
Allow a user to enter a 5-digit U.S. ZIP code and receive:
1. Live NOAA weather alerts for that location with plain-language action steps
2. Recent USGS earthquakes within 100 km in the past 72 hours  
3. Top three long-term climate hazards for that ZIP code from a static JSON source
4. A flood insurance likelihood label (High, Moderate, Low) inferred from FEMA NFHL zones, with a clear disclaimer

Technical Requirements:
- Operate without a database using public APIs + static assets + HTTP caching
- Production-ready for Vercel deployment

## Technology Stack
- Framework: Next.js 14 (App Router - NOT Pages Router)
- Language: TypeScript
- Styling: Tailwind CSS
- API: Next.js Route Handlers under /app/api
- Data Storage: Static JSON under /public/data
- Authentication: None
- Database: None

## Environment Variables
Create .env.local with:
```
NOAA_USER_AGENT="myriskmap (youremail@example.com)"
NEXT_PUBLIC_SITE_URL="http://localhost:3000"
```

Create .env.local.example with the same variables but placeholder values.

## File Structure (Next.js 14 App Router)
```
/app/
├── page.tsx                    (Landing page)
├── zip/[zip]/page.tsx         (Results page)
├── prep/[hazard]/page.tsx     (Static prep pages)
├── methodology/page.tsx       (Static methodology)
├── layout.tsx                 (Root layout)
└── api/
    ├── alerts/route.ts        (NOAA alerts API)
    ├── quakes/route.ts        (USGS earthquakes API)
    ├── risk/route.ts          (Static risk data API)
    └── flood/route.ts         (FEMA flood zones API)

/lib/
└── geo.ts                     (ZIP to coordinates helper)

/components/ui/
├── ZipSearchForm.tsx
├── AlertCard.tsx
├── RiskBar.tsx
├── FloodLikelihoodBadge.tsx
├── PrepLinks.tsx
├── ShareButton.tsx
├── LoadingSkeleton.tsx
├── Header.tsx
├── Footer.tsx
└── Container.tsx

/public/data/
└── risk.zip.min.json         (Static risk scores)
```

## Core Components Requirements

### ZipSearchForm.tsx
- Input field with regex validation: /^\d{5}$/
- Submit functionality that navigates to /zip/[zip]
- Show validation error for invalid ZIP codes
- Use TypeScript interface for props

### AlertCard.tsx  
- Display NOAA alerts with severity badge (color-coded)
- Show headline, action steps, and expiration time
- Handle empty state: "No active alerts"

### RiskBar.tsx
- Horizontal progress bar (0–100 scale)
- Display hazard name, score, and "why" explanation  
- Link to corresponding /prep/[hazard] page

### FloodLikelihoodBadge.tsx
- Display High/Moderate/Low likelihood with color coding
- Show one-sentence rationale
- Include disclaimer text

### LoadingSkeleton.tsx
- Animated placeholder component using Tailwind
- Use for loading states in each section

## ZIP to Coordinates Helper
File: /lib/geo.ts

```typescript
export function isValidZip(zip: string): boolean {
  return /^\d{5}$/.test(zip);
}

export async function getCoordsForZip(zip: string): Promise<{
  lat: number;
  lon: number;
  city?: string;
  state?: string;
}> {
  if (!isValidZip(zip)) {
    throw new Error('Invalid ZIP code format');
  }
  
  const controller = new AbortController();
  const timeout = setTimeout(() => controller.abort(), 8000);
  
  try {
    const response = await fetch(`https://api.zippopotam.us/us/${zip}`, {
      signal: controller.signal
    });
    clearTimeout(timeout);
    
    if (!response.ok) {
      throw new Error('ZIP code not found');
    }
    
    const data = await response.json();
    const place = data.places?.[0];
    
    return {
      lat: parseFloat(place.latitude),
      lon: parseFloat(place.longitude),
      city: place['place name'],
      state: place['state abbreviation'] || place['state']
    };
  } catch (error) {
    clearTimeout(timeout);
    throw error;
  }
}
```

## API Route Handlers (Next.js App Router)

### 1. GET /app/api/alerts/route.ts
```typescript
import { NextRequest, NextResponse } from 'next/server';
import { getCoordsForZip } from '@/lib/geo';

export async function GET(request: NextRequest) {
  try {
    const searchParams = request.nextUrl.searchParams;
    const zip = searchParams.get('zip');
    
    if (!zip) {
      return NextResponse.json({ error: 'ZIP parameter required' }, { status: 400 });
    }

    const coords = await getCoordsForZip(zip);
    
    const controller = new AbortController();
    const timeout = setTimeout(() => controller.abort(), 10000);
    
    const response = await fetch(
      `https://api.weather.gov/alerts/active?point=${coords.lat},${coords.lon}`,
      {
        signal: controller.signal,
        headers: {
          'User-Agent': process.env.NOAA_USER_AGENT || 'myriskmap',
          'Accept': 'application/geo+json'
        }
      }
    );
    clearTimeout(timeout);
    
    if (!response.ok) {
      throw new Error('NOAA API error');
    }
    
    const data = await response.json();
    
    const alerts = data.features?.map((feature: any) => ({
      title: feature.properties?.headline || 'Weather Alert',
      severity: feature.properties?.severity || 'Unknown',
      effective: feature.properties?.effective,
      expires: feature.properties?.expires,
      headline: feature.properties?.headline,
      instructions: feature.properties?.instruction,
      source: 'NOAA NWS'
    })) || [];

    const result = {
      zip,
      lat: coords.lat,
      lon: coords.lon,
      city: coords.city,
      state: coords.state,
      alerts
    };

    return NextResponse.json(result, {
      headers: {
        'Cache-Control': 'public, s-maxage=300, stale-while-revalidate=600'
      }
    });
    
  } catch (error) {
    console.error('Alerts API error:', error);
    return NextResponse.json(
      { error: 'Failed to fetch weather alerts' },
      { status: 500 }
    );
  }
}
```

### 2. GET /app/api/quakes/route.ts
```typescript
import { NextRequest, NextResponse } from 'next/server';
import { getCoordsForZip } from '@/lib/geo';

export async function GET(request: NextRequest) {
  try {
    const searchParams = request.nextUrl.searchParams;
    const zip = searchParams.get('zip');
    
    if (!zip) {
      return NextResponse.json({ error: 'ZIP parameter required' }, { status: 400 });
    }

    const coords = await getCoordsForZip(zip);
    
    // Calculate 72 hours ago
    const startTime = new Date(Date.now() - 72 * 60 * 60 * 1000).toISOString();
    
    const controller = new AbortController();
    const timeout = setTimeout(() => controller.abort(), 10000);
    
    const response = await fetch(
      `https://earthquake.usgs.gov/fdsnws/event/1/query?format=geojson&latitude=${coords.lat}&longitude=${coords.lon}&maxradiuskm=100&starttime=${startTime}`,
      { signal: controller.signal }
    );
    clearTimeout(timeout);
    
    if (!response.ok) {
      throw new Error('USGS API error');
    }
    
    const data = await response.json();
    
    const quakes = data.features
      ?.map((feature: any) => ({
        magnitude: feature.properties?.mag || null,
        timeISO: feature.properties?.time ? new Date(feature.properties.time).toISOString() : null,
        place: feature.properties?.place || 'Unknown location',
        url: feature.properties?.url || ''
      }))
      .sort((a: any, b: any) => (b.magnitude || 0) - (a.magnitude || 0))
      .slice(0, 3) || [];

    const result = {
      zip,
      lat: coords.lat,
      lon: coords.lon,
      quakes
    };

    return NextResponse.json(result, {
      headers: {
        'Cache-Control': 'public, s-maxage=300, stale-while-revalidate=600'
      }
    });
    
  } catch (error) {
    console.error('Earthquakes API error:', error);
    return NextResponse.json(
      { error: 'Failed to fetch earthquake data' },
      { status: 500 }
    );
  }
}
```

### 3. GET /app/api/risk/route.ts
```typescript
import { NextRequest, NextResponse } from 'next/server';
import { promises as fs } from 'fs';
import path from 'path';

export async function GET(request: NextRequest) {
  try {
    const searchParams = request.nextUrl.searchParams;
    const zip = searchParams.get('zip');
    
    if (!zip) {
      return NextResponse.json({ error: 'ZIP parameter required' }, { status: 400 });
    }

    const filePath = path.join(process.cwd(), 'public', 'data', 'risk.zip.min.json');
    const fileContents = await fs.readFile(filePath, 'utf8');
    const riskData = JSON.parse(fileContents);
    
    const result = {
      zip,
      hazards: riskData[zip]?.hazards || []
    };

    return NextResponse.json(result, {
      headers: {
        'Cache-Control': 'public, s-maxage=31536000' // 1 year for static data
      }
    });
    
  } catch (error) {
    console.error('Risk API error:', error);
    return NextResponse.json(
      { error: 'Failed to fetch risk data' },
      { status: 500 }
    );
  }
}
```

### 4. GET /app/api/flood/route.ts
```typescript
import { NextRequest, NextResponse } from 'next/server';
import { getCoordsForZip } from '@/lib/geo';

export async function GET(request: NextRequest) {
  try {
    const searchParams = request.nextUrl.searchParams;
    const zip = searchParams.get('zip');
    
    if (!zip) {
      return NextResponse.json({ error: 'ZIP parameter required' }, { status: 400 });
    }

    const coords = await getCoordsForZip(zip);
    
    const controller = new AbortController();
    const timeout = setTimeout(() => controller.abort(), 15000);
    
    try {
      const response = await fetch(
        `https://hazards.fema.gov/arcgis/rest/services/public/NFHL/MapServer/28/query?where=1%3D1&geometry=${coords.lon},${coords.lat}&geometryType=esriGeometryPoint&spatialRel=esriSpatialRelIntersects&outFields=FLD_ZONE,ZONE_SUBTY&returnGeometry=false&f=json`,
        { signal: controller.signal }
      );
      clearTimeout(timeout);
      
      if (!response.ok) {
        throw new Error('FEMA API error');
      }
      
      const data = await response.json();
      
      // SFHA zones: A, AE, AH, AO, A99, AR, V, VE
      const sfhaZones = ['A', 'AE', 'AH', 'AO', 'A99', 'AR', 'V', 'VE'];
      
      let likelihood: 'High' | 'Moderate' | 'Low' = 'Low';
      let rationale = 'Area shows lower flood risk based on FEMA mapping';
      
      if (data.features && data.features.length > 0) {
        const zones = data.features.map((f: any) => f.attributes?.FLD_ZONE).filter(Boolean);
        const hasSFHA = zones.some((zone: string) => 
          sfhaZones.some(sfha => zone?.startsWith(sfha))
        );
        
        if (hasSFHA) {
          likelihood = 'High';
          rationale = 'Located in Special Flood Hazard Area (SFHA)';
        } else {
          likelihood = 'Moderate';
          rationale = 'Near mapped flood zones but not in high-risk area';
        }
      }

      const result = {
        zip,
        lat: coords.lat,
        lon: coords.lon,
        likelihood,
        rationale,
        disclaimer: 'Informational only. Confirm with local officials or your lender.'
      };

      return NextResponse.json(result, {
        headers: {
          'Cache-Control': 'public, s-maxage=86400, stale-while-revalidate=172800'
        }
      });
      
    } catch (fetchError) {
      clearTimeout(timeout);
      
      // Fallback for FEMA API issues
      const result = {
        zip,
        lat: coords.lat,
        lon: coords.lon,
        likelihood: 'Unknown' as const,
        rationale: 'Flood data temporarily unavailable',
        disclaimer: 'Check with local officials for current flood zone status'
      };

      return NextResponse.json(result, {
        headers: {
          'Cache-Control': 'public, s-maxage=300'
        }
      });
    }
    
  } catch (error) {
    console.error('Flood API error:', error);
    return NextResponse.json(
      { error: 'Failed to fetch flood data' },
      { status: 500 }
    );
  }
}
```

## Static Data File
Create /public/data/risk.zip.min.json:

```json
{
  "11208": {
    "hazards": [
      { "name": "flood", "score": 78, "why": "coastal storms and heavy rainfall" },
      { "name": "heat", "score": 64, "why": "urban heat island effect" },
      { "name": "wind", "score": 52, "why": "nor'easters and tropical remnants" }
    ]
  },
  "33401": {
    "hazards": [
      { "name": "hurricane", "score": 88, "why": "exposure to Atlantic tropical cyclones" },
      { "name": "flood", "score": 72, "why": "storm surge and heavy rain" },
      { "name": "heat", "score": 70, "why": "high summer temperatures and humidity" }
    ]
  },
  "10001": {
    "hazards": [
      { "name": "flood", "score": 62, "why": "intense rainfall and drainage limits" },
      { "name": "heat", "score": 66, "why": "urban heat island effect" },
      { "name": "wind", "score": 50, "why": "winter storms and nor'easters" }
    ]
  },
  "90210": {
    "hazards": [
      { "name": "earthquake", "score": 75, "why": "proximity to active fault lines" },
      { "name": "wildfire", "score": 68, "why": "dry conditions and hillside location" },
      { "name": "heat", "score": 58, "why": "inland valley heat exposure" }
    ]
  },
  "70112": {
    "hazards": [
      { "name": "flood", "score": 95, "why": "below sea level with hurricane storm surge" },
      { "name": "hurricane", "score": 82, "why": "Gulf Coast tropical cyclone exposure" },
      { "name": "heat", "score": 74, "why": "subtropical climate and urban heat" }
    ]
  },
  "80424": {
    "hazards": [
      { "name": "winter", "score": 85, "why": "high elevation mountain snowstorms" },
      { "name": "wind", "score": 62, "why": "mountain winds and chinook events" },
      { "name": "flood", "score": 45, "why": "spring snowmelt and flash floods" }
    ]
  }
}
```

## Pages and User Experience

### Landing Page (/)
- Headline: "See your local climate risk and live alerts"
- Subheadline: "Enter your ZIP code to get instant hazard information, flood insurance likelihood, and safety steps"
- ZIP input field with pattern validation (\d{5})
- Submit functionality that navigates to /zip/[zip]
- Footer citing sources (NOAA NWS, USGS, FEMA NFHL)

### Results Page (/zip/[zip])
- Fetch data from relative API paths: /api/alerts?zip=..., /api/quakes?zip=..., /api/risk?zip=..., /api/flood?zip=...
- Render sections independently with LoadingSkeleton components
- Section 1: Weather Alerts (AlertCard components)
- Section 2: Long-term Risk (RiskBar components)  
- Section 3: Flood Insurance Likelihood (FloodLikelihoodBadge component)
- Section 4: Preparation Links (PrepLinks component)
- Section 5: Share functionality (ShareButton component)
- Handle empty states gracefully: "No active alerts", "No recent earthquakes nearby"
- Soft-fail any section that errors while rendering others

### Preparation Pages (/prep/[hazard])
- Static pages for hazards: flood, earthquake, heat, wind, winter, air, hurricane, wildfire
- 150–200 words of practical tips per hazard
- Clear action items and authoritative source links

### Methodology Page (/methodology)
- Explain data sources, caching strategy, NFHL simplifications
- Include "informational only" disclaimer

## Performance and Caching Requirements
- Lighthouse mobile score target: ≥85
- Cache durations:
  - Alerts/Earthquakes: 300 seconds
  - Flood data: 86400 seconds  
  - Risk data: Static/infinite
- AbortController timeouts:
  - Alerts/Earthquakes: 10 seconds
  - Flood: 15 seconds
  - ZIP lookup: 8 seconds
- Keep API handlers concise and avoid blocking operations

## SEO and Social Media
Set metadata on all pages:
- Title: `Climate Risk & Alerts for ZIP ${zip} | MyRiskMap`
- Description: `Live weather alerts, earthquake data, and long-term climate risk for ZIP code ${zip}. Plus flood insurance likelihood assessment.`

## Deliverables
Complete Next.js 14 repository including:
1. All App Router pages and API handlers as specified
2. /lib/geo.ts helper with proper TypeScript types
3. /public/data/risk.zip.min.json with demo entries
4. All UI components with TypeScript interfaces
5. Tailwind configuration with responsive design
6. .env.local.example with required variables
7. README with setup, deployment, and troubleshooting instructions

## Post-Generation Testing Checklist
1. Run locally: `npm install` then `npm run dev`
2. Test demo ZIP codes: /zip/11208, /zip/33401, /zip/90210, /zip/70112
3. Verify error handling with invalid ZIP codes
4. Test mobile responsiveness
5. Deploy to Vercel and set environment variables
6. Confirm production site renders all data types for multiple regions
