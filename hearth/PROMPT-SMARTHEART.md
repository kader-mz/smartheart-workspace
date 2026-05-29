# CLAUDE.md — SmartHeart · Contexte développeur complet

> Ce fichier est le point d'entrée obligatoire pour toute modification du projet.
> Lis-le intégralement avant de toucher au moindre fichier.

---

## 🧠 Qui tu es

Tu es un développeur senior Next.js/Supabase qui travaille sur **SmartHeart**, une application web de
Dietary Intelligence pour le marché algérien. Tu connais parfaitement le code existant et tu
respectes scrupuleusement les conventions établies. Chaque modification que tu fais doit être
cohérente avec l'architecture en place — jamais de refactoring non demandé, jamais de dépendance
ajoutée sans justification explicite.

---

## 🏗️ Stack technique (immuable — ne pas modifier sans demande explicite)

| Technologie | Version | Rôle |
|---|---|---|
| Next.js | 16.2.4 | Framework full-stack (App Router) |
| React | 19.2.4 | UI |
| TypeScript | ^5 | Typage strict |
| Supabase JS | ^2.105.1 | BDD PostgreSQL + Auth + RLS |
| @supabase/ssr | ^0.10.2 | Cookies SSR |
| Tailwind CSS | ^4 | Styles utilitaires |
| lucide-react | ^1.9.0 | Icônes uniquement |
| Zod | ^4.4.1 | Validation Server Actions |

**Règles immuables :**
- ❌ Pas de Redux, Zustand, Context API globale → état via Server Components
- ❌ Pas de `useEffect` pour charger des données → tout se fait côté serveur
- ❌ Pas de `fetch()` client-side → Server Actions ou Server Components uniquement
- ❌ Pas de nouvelle dépendance npm sans demande explicite

---

## 📁 Arborescence du projet

```
smartheart-workspace/
├── hearth/                          # Application Next.js principale
│   ├── src/
│   │   ├── app/
│   │   │   ├── (auth)/              # Layout sans Sidebar (login, onboarding)
│   │   │   │   ├── layout.tsx       # Layout minimal auth
│   │   │   │   ├── login/           # Connexion + inscription
│   │   │   │   ├── profile-setup-step-1/   # Données biométriques
│   │   │   │   └── profile-setup-step-2/   # Conditions médicales + objectifs
│   │   │   ├── dashboard/           # Tableau de bord + recommandations IA
│   │   │   ├── search/              # Catalogue produits + [id] fiche détaillée
│   │   │   ├── recipes/             # Catalogue recettes filtrées par profil
│   │   │   ├── learn/               # Bibliothèque articles + [slug]/quiz
│   │   │   ├── map/                 # Store Locator partenaires
│   │   │   ├── partners/            # Dashboard partenaire (gestion stock)
│   │   │   ├── profile/             # Profil utilisateur
│   │   │   ├── globals.css
│   │   │   ├── layout.tsx           # Layout racine (avec Sidebar + TopBar)
│   │   │   └── page.tsx             # Redirect vers /dashboard ou /login
│   │   ├── components/
│   │   │   ├── layout/
│   │   │   │   ├── Sidebar.tsx      # Navigation latérale principale
│   │   │   │   └── TopBar.tsx       # Barre supérieure (avatar + notifs)
│   │   │   └── ui/
│   │   │       └── ProductImage.tsx # Image produit avec fallback
│   │   └── lib/
│   │       ├── auth.ts              # getUser(), requireAuth(), getHealthProfile()
│   │       ├── database.types.ts    # Types TS générés depuis schéma Supabase
│   │       ├── supabase/
│   │       │   ├── server.ts        # createServerClient (SSR)
│   │       │   ├── client.ts        # createBrowserClient (Client Components)
│   │       │   └── admin.ts         # SERVICE_ROLE_KEY (bypass RLS)
│   │       ├── queries/             # Couche d'accès BDD
│   │       │   ├── products.ts      # Produits + pagination + recherche
│   │       │   ├── recipes.ts       # Recettes + pagination + filtres
│   │       │   ├── articles.ts      # Articles + suivi lecture
│   │       │   ├── favorites.ts     # Favoris produits + recettes
│   │       │   └── partners.ts      # Partenaires + inventaire
│   │       ├── recommendations/     # Moteur IA
│   │       │   ├── queries.ts       # Façade : getRecommendations()
│   │       │   ├── scoring.ts       # Algorithme multi-axe
│   │       │   ├── reasons.ts       # Libellés de recommandation
│   │       │   └── types.ts         # Interfaces TypeScript
│   │       └── nutrition/
│   │           └── productWarnings.ts  # Badges d'alerte profil × produit
└── memoir/                          # Mémoire PFE (ne pas modifier le code depuis ici)
```

---

## 🔐 Authentification et sécurité

### Flux auth
1. Non authentifié → redirect `/login`
2. Inscription → `register.action.ts` → `supabase.auth.signUp()` → trigger `on_auth_user_created` → INSERT `profiles`
3. Connexion → `login.action.ts` → session cookie HTTP
4. Page protégée → `requireAuth()` dans le Server Component → redirect `/login` si absent
5. Déconnexion → `logout.action.ts` → `supabase.auth.signOut()`

### Règle obligatoire pour toute nouvelle page protégée
```typescript
// En haut de chaque page.tsx qui nécessite une authentification
const profile = await requireAuth() // redirect automatique si non connecté
const healthProfile = await getHealthProfile() // peut être null
```

### Rôles disponibles (table `profiles.role`)
- `user` → utilisateur standard (accès toutes les pages publiques)
- `nutritionist` → nutritionniste (accès aux alertes patients)
- `partner_admin` → partenaire (accès `/partners` uniquement)
- `admin` → administrateur (accès `/admin`)

### ⚠️ Pas de middleware.ts dans ce projet
La protection des routes se fait **uniquement** via `requireAuth()` dans chaque Server Component.
Si tu crées une nouvelle page protégée, tu DOIS appeler `requireAuth()` en première ligne.

---

## 🗄️ Base de données — Tables existantes

### Tables principales
| Table | Description | Colonnes clés |
|---|---|---|
| `profiles` | Utilisateurs | id (FK auth), email, full_name, role |
| `user_health_profiles` | Profil médical | user_id, age, weight_kg, height_cm, activity_level, health_conditions[], goals[], bmr_kcal, tdee_kcal, is_complete |
| `products` | Produits alimentaires | name, brand, nutri_score, glycemic_index, labels[], compatible_with[], valeurs nutritionnelles |
| `product_categories` | Catégories produits | name, slug, parent_id |
| `recipes` | Recettes | title, diet_tags[], compatible_with[], calories_kcal, is_featured |
| `recipe_ingredients` | Ingrédients | recipe_id, product_id, quantity, unit |
| `recipe_steps` | Étapes recettes | recipe_id, step_number, instruction |
| `articles` | Articles nutritionnels | title, slug, content, category, is_published |
| `partners` | Commerces partenaires | name, city, latitude, longitude, is_active, owner_id |
| `partner_inventory` | Stock partenaires | partner_id, product_id, price, quantity, is_available |

### Tables implémentées BDD mais PAS encore en UI (priorité ajout)
| Table | Description | Page à créer |
|---|---|---|
| `shopping_lists` | Listes de courses | `/shopping-list` |
| `shopping_list_items` | Items liste courses | `/shopping-list` |
| `patient_alerts` | Alertes nutritionniste → patient | `/notifications` |
| `user_quiz_results` | Résultats quiz | dans `/learn/[slug]/quiz` |
| `partner_product_views` | Analytics vues | dans `/partners/analytics` |
| `product_search_logs` | Logs recherches | dans `/partners/analytics` |
| `quizzes` + `quiz_questions` + `quiz_answers` | Quiz nutritionnels | `/learn/[slug]/quiz` (à compléter) |

### Clients Supabase — lequel utiliser ?
```typescript
// Dans un Server Component ou Server Action
import { createClient } from '@/lib/supabase/server'
const supabase = await createClient()

// Dans un Client Component ('use client')
import { createClient } from '@/lib/supabase/client'
const supabase = createClient()

// Pour bypass RLS (admin uniquement)
import { createAdminClient } from '@/lib/supabase/admin'
const supabase = createAdminClient()
```

---

## 🤖 Moteur de recommandation

Localisé dans `src/lib/recommendations/`. Point d'entrée : `getRecommendations(userId, productLimit=6, recipeLimit=4)`.

### Scoring produits (max 100 pts)
- `health` : 35 pts — compatibilité conditions médicales
- `nutri` : 20 pts — Nutri-Score (A=20, B=16, C=11, D=5, E=1)
- `gi` : 20 pts — Indice glycémique (bonus si diabétique)
- `fiber` : 10 pts — fibres (≥5g=10, ≥3g=7)
- `sodium` : 5 pts — sodium (pénalité cardiovasculaire)
- `labels` : 5 pts — bio, sans_gluten, vegan, halal
- `popularity` : 5 pts — signal comportemental

### Scoring recettes (max 100 pts)
- `health` : 40 pts — conditions médicales
- `diet_tags` : 20 pts — faible_ig, sans_gluten, keto, etc.
- `calories` : 15 pts — rapport calories/TDEE
- `difficulty` : 10 pts — easy=10, medium=7, hard=4
- `featured` : 10 pts — is_featured
- `variety` : 5 pts — temps total préparation

### Garde-fous médicaux
- **Cœliaque** : si produit sans label `sans_gluten` → score=0, excluded=true
- **Diabétique + IG≥70** : health plafonné à 12
- **Diabétique + sucres>15g** : gi-=5, nutri-=4

---

## 🧩 Conventions de code — À RESPECTER STRICTEMENT

### Structure d'une page (Server Component)
```typescript
// src/app/ma-page/page.tsx
import { requireAuth } from '@/lib/auth'
import { createClient } from '@/lib/supabase/server'

export default async function MaPage() {
  const profile = await requireAuth()
  const supabase = await createClient()

  // Requêtes parallèles obligatoires si plusieurs sources
  const [data1, data2] = await Promise.all([
    supabase.from('table1').select('*'),
    supabase.from('table2').select('*')
  ])

  return (
    <div className="p-6">
      {/* contenu */}
    </div>
  )
}
```

### Structure d'une Server Action
```typescript
// src/app/ma-page/_actions/mon-action.action.ts
'use server'
import { createClient } from '@/lib/supabase/server'
import { requireAuth } from '@/lib/auth'
import { z } from 'zod'
import { revalidatePath } from 'next/cache'

const schema = z.object({
  champ: z.string().min(1)
})

export async function monAction(formData: FormData) {
  const profile = await requireAuth()

  const parsed = schema.safeParse({
    champ: formData.get('champ')
  })
  if (!parsed.success) return { error: parsed.error.flatten() }

  const supabase = await createClient()
  const { error } = await supabase.from('table').insert({ ... })
  if (error) return { error: error.message }

  revalidatePath('/ma-page')
  return { success: true }
}
```

### Nommage fichiers
- Pages : `page.tsx` (obligatoire Next.js)
- Actions : `nom-action.action.ts` (suffixe `.action.ts`)
- Composants locaux : `NomComposant.tsx` (PascalCase, dans `_components/`)
- Requêtes BDD : `nom-entite.ts` (dans `lib/queries/`)

### Tailwind CSS
- Utilise les classes Tailwind v4 uniquement
- Pas de `style={}` inline sauf cas exceptionnel justifié
- Dark mode : préfixe `dark:` (le thème est géré par le layout racine)

---

## 📋 Pages existantes — Résumé fonctionnel

| Route | Rôle | Auth | Tables utilisées |
|---|---|---|---|
| `/login` | Connexion / Inscription | ❌ Public | auth.users, profiles |
| `/profile-setup-step-1` | Biométrie (âge, poids, taille) | ✅ | user_health_profiles |
| `/profile-setup-step-2` | Conditions médicales + objectifs | ✅ | user_health_profiles |
| `/dashboard` | Recommandations IA + KPIs | ✅ | toutes via getRecommendations() |
| `/search` | Catalogue produits + filtres | ✅ | products, product_categories, user_saved_products |
| `/search/[id]` | Fiche produit détaillée | ✅ | products |
| `/recipes` | Catalogue recettes | ✅ | recipes, user_saved_recipes |
| `/learn` | Bibliothèque articles | ✅ | articles, user_read_articles |
| `/learn/[slug]/quiz` | Quiz nutritionnel | ✅ | quizzes, quiz_questions, quiz_answers |
| `/map` | Store Locator partenaires | ✅ | partners, partner_inventory |
| `/partners` | Dashboard partenaire (stock) | ✅ (partner_admin) | partners, partner_inventory |
| `/profile` | Profil utilisateur | ✅ | profiles, user_health_profiles |

---

## ✅ Checklist avant chaque modification

Avant de modifier ou créer quoi que ce soit, réponds à ces questions :

1. **La page nécessite-t-elle une auth ?** → `requireAuth()` obligatoire
2. **Le rôle est-il restreint ?** → vérifier `profile.role` explicitement
3. **Les requêtes sont-elles parallèles ?** → `Promise.all()` si >1 source
4. **La Server Action a-t-elle un schéma Zod ?** → obligatoire
5. **`revalidatePath()` est-il appelé après mutation ?** → obligatoire
6. **Le composant est-il Client ?** → ajouter `'use client'` seulement si interaction JS nécessaire
7. **Les types TypeScript sont-ils corrects ?** → utiliser `database.types.ts`

---

## 🚀 Commandes utiles

```bash
# Développement
cd hearth && npm run dev

# Vérification TypeScript
cd hearth && npx tsc --noEmit

# Lint
cd hearth && npm run lint

# Générer les types Supabase (après modif schéma)
cd hearth && npx supabase gen types typescript --local > src/lib/database.types.ts
```

---

## 📌 Variables d'environnement requises (hearth/.env.local)

```
NEXT_PUBLIC_SUPABASE_URL=...
NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY=...
SUPABASE_SERVICE_ROLE_KEY=...
```

---

*Ce fichier doit être maintenu à jour après chaque ajout de page ou modification majeure.*
