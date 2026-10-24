# Dynamic Sitemap for Supabase + Cloudflare KV

Generateur de sitemap XML dynamique qui lit les URLs depuis une table Supabase et les sert via Cloudflare Workers. Gere automatiquement le sitemap index pour les sites de plus de 50 000 URLs.

## Fonctionnalites

- Sitemap XML conforme au protocole sitemaps.org
- Sitemap index automatique au-dela de 50 000 URLs
- Pagination via une RPC Supabase (`sitemap_pages_paginated`)
- Cache Cloudflare pour les performances
- Compatible avec Google Search Console, Bing Webmaster Tools

## Comment ca marche

1. Les URLs sont stockees dans une table `sitemap_urls` (alimentee par un trigger sur `domain_pages`)
2. L'Edge Function Supabase genere le XML a la volee
3. Le Worker Cloudflare sert `/sitemap.xml` et `/sitemap-N.xml`

## Schema

```sql
CREATE TABLE sitemap_urls (
  id SERIAL PRIMARY KEY,
  domain TEXT NOT NULL,
  path TEXT NOT NULL,
  lastmod TIMESTAMPTZ DEFAULT now(),
  changefreq TEXT DEFAULT 'weekly',
  priority NUMERIC DEFAULT 0.7,
  submitted_to_gsc BOOLEAN DEFAULT false,
  UNIQUE(domain, path)
);

-- Trigger : chaque INSERT/UPDATE sur domain_pages → upsert sitemap_urls
CREATE OR REPLACE FUNCTION sync_sitemap_url()
RETURNS TRIGGER AS $$
BEGIN
  INSERT INTO sitemap_urls (domain, path, lastmod)
  VALUES (NEW.domain, NEW.slug, now())
  ON CONFLICT (domain, path) DO NOTHING;
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

## RPC paginee

```sql
CREATE OR REPLACE FUNCTION sitemap_pages_paginated(
  p_domain TEXT,
  p_page INT DEFAULT 1,
  p_per_page INT DEFAULT 50000
)
RETURNS TABLE(path TEXT, lastmod TEXT) AS $$
BEGIN
  RETURN QUERY
  SELECT su.path, to_char(su.lastmod, 'YYYY-MM-DD') as lastmod
  FROM sitemap_urls su
  WHERE su.domain = p_domain
  ORDER BY su.lastmod DESC
  LIMIT p_per_page
  OFFSET (p_page - 1) * p_per_page;
END;
$$ LANGUAGE plpgsql;
```

## Deploiement

L'Edge Function se deploie via la CLI Supabase :

```bash
supabase functions deploy sitemap --project-ref YOUR_PROJECT_REF --use-api
```

## En production

Ce systeme gere les sitemaps de 119 domaines pour [BusinessDigital.fr](https://businessdigital.fr), avec plus de 5,7 millions d'URLs indexees. Le trigger automatique garantit que chaque nouvelle page apparait dans le sitemap sans intervention manuelle.

Decouvrez nos outils et formations IA : [businessdigital.fr/catalogue-formations](https://businessdigital.fr/catalogue-formations/).

## Licence

MIT
