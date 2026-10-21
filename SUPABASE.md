# Voorbereiding: online opslag met Supabase (optie 2)

Dit is nog **niet actief**. De app slaat nu alles op je telefoon op. Dit document beschrijft wat er al klaarligt en wat er nog moet gebeuren als je later wilt synchroniseren.

## Wat er al klaarligt in de app
- **Vaste ID's:** elk record (oefening, schema, training) heeft een unieke id die op elk apparaat hetzelfde blijft. De standaardoefeningen hebben vaste id's (`lib-bankdrukken`, enz.).
- **Tijdstempels:** elk record heeft `updatedAt`. Bij een wijziging wordt die bijgewerkt.
- **Zacht verwijderen:** verwijderen zet `deleted: true` in plaats van het record te wissen, zodat een verwijdering ook naar andere apparaten kan.
- **Samenvoegen:** de functie `mergeData()` voegt twee datasets samen per record; de nieuwste `updatedAt` wint. Die wordt nu gebruikt bij het terugzetten van een back-up en kan straks ook bij het synchroniseren worden gebruikt.
- **Aparte opslaglaag:** het scherm praat alleen met `LocalAdapter` (`load` / `save`). Een online laag kan daarnaast komen zonder dat de schermen veranderen.

## Datamodel
```
meta      { schemaVersion, deviceId, createdAt, lastBackupAt, activeWorkoutId }   // alleen lokaal
settings  { id: 'settings', defaultRest, keepAwake, theme, updatedAt }
exercises [{ id, name, muscle, equipment, custom, updatedAt, deleted }]
templates [{ id, name, items: [{ id, exerciseId, warmups, sets, reps, restSec }], updatedAt, deleted }]
workouts  [{ id, templateId, name, startedAt, endedAt, updatedAt, deleted,
             exercises: [{ id, exerciseId, note, targetReps, restSec,
                           sets: [{ id, kg, reps, warmup, done }] }] }]
```
Een training wordt als één geheel opgeslagen (inclusief oefeningen en sets). Dat houdt synchroniseren eenvoudig.

## Wat je zelf doet (± 10 minuten)
1. Maak een gratis account op supabase.com en een nieuw project (regio: West EU).
2. Ga naar **SQL Editor** en voer het script hieronder uit.
3. Ga naar **Authentication › Providers** en laat **Email** aanstaan (inloggen met een link per e-mail).
4. Geef mij de **Project URL** en de **anon public key** (Project Settings › API). Deze sleutel mag in de app staan; de beveiliging zit in de regels hieronder.

## SQL-script
```sql
create table public.records (
  id          text        not null,
  user_id     uuid        not null default auth.uid() references auth.users(id) on delete cascade,
  collection  text        not null check (collection in ('settings','exercises','templates','workouts')),
  data        jsonb       not null,
  updated_at  timestamptz not null,
  deleted     boolean     not null default false,
  primary key (user_id, collection, id)
);

create index records_user_updated on public.records (user_id, updated_at);

alter table public.records enable row level security;

create policy "Eigen records lezen"     on public.records for select using (auth.uid() = user_id);
create policy "Eigen records toevoegen" on public.records for insert with check (auth.uid() = user_id);
create policy "Eigen records wijzigen"  on public.records for update using (auth.uid() = user_id) with check (auth.uid() = user_id);
```

## Hoe het synchroniseren straks werkt
1. Inloggen met je e-mailadres (eenmalig per apparaat).
2. Bij openen en na elke afgeronde training: stuur alle records met `updatedAt` na de laatste sync naar Supabase (`upsert`).
3. Haal alle records op met `updated_at` na de laatste sync en voeg ze samen met `mergeData()`.
4. Sla het tijdstip van deze sync op in `meta.lastSyncAt`.

Zonder internet blijft de app gewoon werken; bij de volgende keer online wordt bijgewerkt. De back-upknop blijft bestaan als extra zekerheid.
