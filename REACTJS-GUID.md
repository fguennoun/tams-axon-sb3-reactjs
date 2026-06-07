# REACTJS-GUID.md — Guide React.js & Redux Toolkit (TAMS)

## Table des Matières

1. [Introduction](#1-introduction)
2. [Composants Fonctionnels & JSX](#2-composants-fonctionnels--jsx)
3. [Props (Propriétés)](#3-props-propriétés)
4. [Hooks d'État : useState](#4-hooks-détat--usestate)
5. [Hooks d'Effet : useEffect](#5-hooks-deffet--useeffect)
6. [React Router DOM](#6-react-router-dom)
7. [Redux Toolkit — Store Central](#7-redux-toolkit--store-central)
8. [Redux Toolkit — Slices (State + Reducers)](#8-redux-toolkit--slices-state--reducers)
9. [Redux Toolkit — Async Thunks (Appels API)](#9-redux-toolkit--async-thunks-appels-api)
10. [Services Axios (HTTP Client)](#10-services-axios-http-client)
11. [Hooks Redux : useSelector, useDispatch](#11-hooks-redux--useselector-usedispatch)
12. [Conditional Rendering](#12-conditional-rendering)
13. [Gestion des Événements & Formulaires](#13-gestion-des-événements--formulaires)
14. [Flux de Données Complet](#14-flux-de-données-complet)
15. [Bonnes Pratiques & Patterns](#15-bonnes-pratiques--patterns)

---

## 1. Introduction

### Stack technique du frontend TAMS

| Technologie | Version | Rôle |
|-------------|---------|------|
| **React** | 18.2.0 | Bibliothèque UI (composants, hooks) |
| **Redux Toolkit** | 1.9.7 | State management centralisé |
| **React Router DOM** | 6.21.3 | Navigation SPA (Single Page Application) |
| **Axios** | 1.6.5 | HTTP client pour appels REST |
| **React Toastify** | 10.0.4 | Notifications utilisateur |
| **React Icons** | 5.0.1 | Icônes UI |

### Architecture du projet

```
tams-uat-front-end-react/
  src/
    app/
      store.js              ← Configuration Redux Store
    components/
      Header.jsx            ← Barre de navigation
      BackButton.jsx        ← Bouton retour réutilisable
      LoadingSpinner.jsx    ← Indicateur de chargement
      Hiring-Manager-Portal/
        HiringManagerPortalHome.jsx
        CreateTalentRequestForm.jsx
        TalentRequestsView.jsx
        TalentRequestByIdView.jsx
        TalentRequestItem.jsx
      Talent-Acquisition-Portal/
        TalentFulfillmentPortalHome.jsx
        TalentFulfillmentsView.jsx
        TalentFulfillmentByIdView.jsx
        TalentFulfillmentItem.jsx
        UpdateAndApproveTalentFulfillmentByIdView.jsx
        ApprovedTalentFulfillmentByIdView.jsx
      Career-Portal/
        CareerPortalHome.jsx
        JobPostsView.jsx
        JobPostByIdView.jsx
        JobPostItem.jsx
    features/
      TalentRequest/
        talentRequestSlice.js    ← State + thunks pour TalentRequest
        talentRequestService.js  ← Appels Axios vers l'API
      TalentFulfillment/
        talentFulfillmentSlice.js
        talentFulfillmentService.js
      CareerPortal/
        careerPortalSlice.js
        careerPortalService.js
    pages/
      TAMSHome.jsx              ← Page d'accueil
    App.js                      ← Configuration routes + layout
    index.js                    ← Point d'entrée (Provider Redux)
    index.css                   ← Styles globaux
```

---

## 2. Composants Fonctionnels & JSX

### Concept

Un **composant fonctionnel** est une fonction JavaScript qui retourne du JSX (HTML dans du JS). React 18 privilégie les fonctions aux classes.

### Exemple dans TAMS

```jsx
// src/components/BackButton.jsx
import { Link } from "react-router-dom";

function BackButton({ url, buttonName }) {
  return (
    <Link to={url} className="btn btn-back">
      {buttonName}
    </Link>
  );
}

export default BackButton;
```

**Points clés :**
- Fonction pure : `function BackButton(...)` retourne du JSX
- **Props en paramètre** : `{ url, buttonName }` (destructuration)
- **JSX** : mélange HTML + expressions JS `{buttonName}`
- **Export** : `export default` rend le composant importable

### Arbre des composants TAMS

```
<App>
  <Router>
    <Header />
    <Routes>
      <Route path="/" element={<TAMSHome />} />
      <Route path="/hiring-manager" element={<HiringManagerPortalHome />} />
      <Route path="/create-talent-request" element={<CreateTalentRequestForm />} />
      <Route path="/get-all-talent-requests" element={<TalentRequestsView />} />
      <Route path="/talent-request/:id" element={<TalentRequestByIdView />} />
      <Route path="/talent-fulfillment" element={<TalentFulfillmentPortalHome />} />
      <Route path="/get-all-talent-fulfillment-requests" element={<TalentFulfillmentsView />} />
      <Route path="/talent-fulfillment/:id" element={<TalentFulfillmentByIdView />} />
      <Route path="/career-portal" element={<CareerPortalHome />} />
      <Route path="/career-portal/job-posts" element={<JobPostsView />} />
      <Route path="/career-portal/job-post/:id" element={<JobPostByIdView />} />
    </Routes>
  </Router>
  <ToastContainer />    ← Notifications globales
</App>
```

---

## 3. Props (Propriétés)

### Concept

Les **props** sont des arguments passés d'un parent à un enfant. Elles sont **read-only** (immutables).

### Exemple dans TAMS

```jsx
// Parent : TalentRequestsView.jsx
{talentRequests.map((talentRequest) => (
  <TalentRequestItem
    key={talentRequest.talentRequestId}     // ← prop key (obligatoire dans les listes)
    talentRequest={talentRequest}           // ← prop personnalisée
  />
))}

// Enfant : TalentRequestItem.jsx
function TalentRequestItem({ talentRequest }) {
  return (
    <div className="talent-request">
      <div>{talentRequest.talentRequestTitle}</div>     {/* ← utilisation de la prop */}
      <div>{new Date(talentRequest.startDate).toLocaleDateString("en-US", { timeZone: "UTC" })}</div>
      <div className={`status status-${talentRequest.requestStatus}`}>
        {talentRequest.requestStatus}
      </div>
      <Link to={`/talent-request/${talentRequest.talentRequestId}`} className="btn btn-sm">
        View Talent Request
      </Link>
    </div>
  );
}
```

**Autres exemples de props dans TAMS :**

```jsx
// BackButton — props simples (string)
<BackButton url="/get-all-talent-requests" buttonName="Back to Talent Requests" />

// TalentFulfillmentByIdView — prop objet
<ApprovedTalentFulfillmentByIdView approvedTalentFulfillment={talentFulfillment} />
<UpdateAndApproveTalentFulfillmentByIdView assignedTalentFulfillment={talentFulfillment} isLoading={isLoading} />
```

---

## 4. Hooks d'État : useState

### Concept

`useState` permet d'ajouter un état local à un composant fonctionnel. Retourne un tableau `[valeur, setter]`.

### Exemple dans TAMS

```jsx
// CreateTalentRequestForm.jsx
import { useState } from "react";

function CreateTalentRequestForm() {
  const [talentRequest, setTalentRequest] = useState({
    talentRequestTitle: "",
    jobDescription: { responsibilities: "", qualifications: "" },
    candidateSkills: { coreSkill: "", skillLevel: "" },
    startDate: "",
  });

  // Mise à jour d'un champ simple
  const onChange = (event) =>
    setTalentRequest({
      ...talentRequest,                  // ← spread pour garder les autres champs
      [event.target.name]: event.target.value,
    });

  // Mise à jour d'un champ imbriqué (jobDescription)
  const onChangeJobDescription = (event) =>
    setTalentRequest({
      ...talentRequest,
      jobDescription: {
        ...talentRequest.jobDescription,   // ← spread pour garder l'autre sous-champ
        [event.target.name]: event.target.value,
      },
    });
}
```

```jsx
// UpdateAndApproveTalentFulfillmentByIdView.jsx
const [talentFulfillment, setTalentFulfillment] = useState({
  talentFulfillmentId: assignedTalentFulfillment?.talentFulfillmentId,
  talentRequestId: assignedTalentFulfillment?.talentRequestId,
  talentRequestTitle: assignedTalentFulfillment?.talentRequestTitle,
  requestStatus: "",
  jobDescription: { responsibilities: "", qualifications: "" },
  candidateSkills: { coreSkill: "", skillLevel: "" },
  roleLevel: "",
  employmentType: "",
});
```

**Règles des hooks :**
- Appelés au top niveau du composant (pas dans des boucles/conditions)
- `...` (spread operator) pour l'immutabilité — toujours créer un nouvel objet
- `[event.target.name]` (computed property name) pour mettre à jour dynamiquement le bon champ

---

## 5. Hooks d'Effet : useEffect

### Concept

`useEffect` exécute du code après le rendu du composant. Utilisé pour :
- Appels API au montage
- Souscriptions / nettoyages
- Réactions aux changements de props/state

### Exemple dans TAMS

```jsx
// TalentRequestsView.jsx
import { useEffect } from "react";
import { useSelector, useDispatch } from "react-redux";
import { getAllTalentRequests, reset } from "../../features/TalentRequest/talentRequestSlice";

function TalentRequestsView() {
  const dispatch = useDispatch();

  // Effet au montage : charger les données
  useEffect(() => {
    dispatch(getAllTalentRequests());    // ← dispatch l'async thunk
  }, [dispatch]);                         // ← dépendance : dispatch (stable)

  // Effet de nettoyage au démontage
  useEffect(() => {
    return () => {
      if (isSuccess) {
        dispatch(reset());               // ← reset du state Redux
      }
    };
  }, [dispatch, isSuccess]);

  // ...
}
```

```jsx
// TalentRequestByIdView.jsx
useEffect(() => {
  if (isError) {
    toast.error(message);                // ← notification si erreur
  }
  dispatch(getTalentRequestById(talentRequestId));
}, [isError, message, talentRequestId, dispatch]);
```

| Dépendances (`[]`) | Comportement |
|--------------------|-------------|
| `[]` (vide) | Exécuté **une seule fois** au montage |
| `[dispatch]` | Exécuté au montage (dispatch ne change jamais) |
| `[talentRequestId]` | Exécuté au montage + chaque fois que `talentRequestId` change |
| `[isError, message]` | Exécuté quand l'erreur ou le message changent |

---

## 6. React Router DOM

### Concept

React Router permet la navigation entre les pages sans rechargement du navigateur (SPA).

### Configuration dans TAMS

```jsx
// src/App.js
import { BrowserRouter as Router, Routes, Route } from "react-router-dom";

function App() {
  return (
    <Router>
      <div className="container">
        <Header />
        <Routes>
          <Route path="/" element={<TAMSHome />} />
          <Route path="/hiring-manager" element={<HiringManagerPortalHome />} />
          <Route path="/talent-request/:talentRequestId" element={<TalentRequestByIdView />} />
          <Route path="/career-portal/job-post/:jobPostId" element={<JobPostByIdView />} />
          {/* ... autres routes */}
        </Routes>
      </div>
    </Router>
  );
}
```

### Hooks de routing utilisés

| Hook | Usage dans TAMS | Exemple |
|------|----------------|---------|
| `useParams()` | Récupérer le paramètre d'URL | `const { talentRequestId } = useParams();` |
| `useNavigate()` | Navigation programmatique | `const navigate = useNavigate(); // ... navigate("/get-all-talent-requests")` |
| `Link` | Navigation déclarative | `<Link to="/create-talent-request">Create</Link>` |

### Navigation programmatique vs déclarative

```jsx
// Navigation déclarative (Link dans le JSX)
<Link to={`/talent-request/${talentRequest.talentRequestId}`} className="btn btn-sm">
  View Talent Request
</Link>

// Navigation programmatique (après une action)
const navigate = useNavigate();

const onSubmit = (e) => {
  e.preventDefault();
  dispatch(createTalentRequest(talentRequest));
  // Après succès, rediriger (handle dans useEffect + isSuccess)
};

useEffect(() => {
  if (isSuccess) {
    dispatch(reset());
    navigate("/get-all-talent-requests");   // ← redirection
  }
});
```

---

## 7. Redux Toolkit — Store Central

### Concept

Le **Store** Redux est un objet JavaScript qui contient l'état global de l'application. Avec RTK, on le configure via `configureStore()`.

### Store TAMS

```js
// src/app/store.js
import { configureStore } from "@reduxjs/toolkit";
import talentRequestReducer from "../features/TalentRequest/talentRequestSlice";
import talentFulfillmentReducer from "../features/TalentFulfillment/talentFulfillmentSlice";
import jobPostReducer from "../features/CareerPortal/careerPortalSlice";

export const store = configureStore({
  reducer: {
    talentRequests: talentRequestReducer,       // ← slice name → reducer
    talentFulfillments: talentFulfillmentReducer,
    jobPosts: jobPostReducer,
  },
});
```

### Injection dans React

```jsx
// src/index.js
import { Provider } from "react-redux";
import { store } from "./app/store";

root.render(
  <React.StrictMode>
    <Provider store={store}>     {/* ← rend le store accessible à tous les composants */}
      <App />
    </Provider>
  </React.StrictMode>
);
```

**Structure de l'état global :**
```js
{
  talentRequests: {
    talentRequests: [],       // Liste
    talentRequest: {},        // Élément sélectionné
    isError: false,
    isSuccess: false,
    isLoading: false,
    message: "",
  },
  talentFulfillments: { ... }, // Même structure
  jobPosts: { ... },           // Même structure
}
```

---

## 8. Redux Toolkit — Slices (State + Reducers)

### Concept

Un **slice** est un morceau du store Redux qui contient :
- `name` : nom du slice
- `initialState` : état initial
- `reducers` : fonctions synchrones pour modifier l'état
- `extraReducers` : fonctions pour gérer les actions asynchrones (thunks)

### Exemple TAMS

```js
// src/features/TalentRequest/talentRequestSlice.js
import { createSlice, createAsyncThunk } from "@reduxjs/toolkit";

// État initial
const initialState = {
  talentRequests: [],
  talentRequest: {},
  isError: false,
  isSuccess: false,
  isLoading: false,
  message: "",
};

// Création du slice
export const talentRequestSlice = createSlice({
  name: "talentRequest",
  initialState,
  reducers: {
    // Reducer synchrone : réinitialise l'état
    reset: (state) => initialState,
  },
  extraReducers: (builder) => {
    builder
      // Cas PENDING : chargement en cours
      .addCase(getAllTalentRequests.pending, (state) => {
        state.isLoading = true;
      })
      // Cas FULFILLED : succès
      .addCase(getAllTalentRequests.fulfilled, (state, action) => {
        state.isSuccess = true;
        state.talentRequests = action.payload;     // ← données de l'API
      })
      // Cas REJECTED : erreur
      .addCase(getAllTalentRequests.rejected, (state, action) => {
        state.isError = true;
        state.message = action.payload;            // ← message d'erreur
      });
  },
});

export const { reset } = talentRequestSlice.actions;     // ← actions exportées
export default talentRequestSlice.reducer;               // ← reducer exporté
```

### Les 3 états d'un async thunk

| État | Signification | Chargement dans l'UI |
|------|--------------|---------------------|
| **pending** | L'appel API est en cours | Afficher `<LoadingSpinner />` |
| **fulfilled** | L'appel a réussi | Afficher les données |
| **rejected** | L'appel a échoué | Afficher `toast.error(message)` |

---

## 9. Redux Toolkit — Async Thunks (Appels API)

### Concept

Un **async thunk** est une action asynchrone qui dispatch automatiquement `pending`, `fulfilled` ou `rejected` selon le résultat de l'appel API.

### Cycle de vie d'un thunk

```js
// Définition du thunk
export const createTalentRequest = createAsyncThunk(
  "talentRequests/createTalentRequest",          // ← nom de l'action
  async (talentRequest, thunkAPI) => {            // ← payload + thunkAPI
    try {
      return await talentRequestService.createTalentRequest(talentRequest);  // ← succès → action.fulfilled
    } catch (error) {
      const message =                              // ← échec → action.rejected
        (error.response && error.response.data && error.response.data.message) ||
        error.message ||
        error.toString();
      return thunkAPI.rejectWithValue(message);    // ← payload du rejected
    }
  }
);
```

### Les 3 thunks du TAMS

| Slice | Thunks | Méthode HTTP |
|-------|--------|--------------|
| `talentRequestSlice` | `createTalentRequest`, `getAllTalentRequests`, `getTalentRequestById` | POST, GET, GET |
| `talentFulfillmentSlice` | `getAllTalentFulfillments`, `getTalentFulfillmentById`, `approveTalentFulfillmentJobPost` | GET, GET, POST |
| `jobPostSlice` | `getAllJobPosts`, `getJobPostById` | GET, GET |

### Dispatch d'un thunk depuis un composant

```jsx
const dispatch = useDispatch();

// Avec payload
dispatch(createTalentRequest(talentRequest));          // ← thunk avec payload
dispatch(approveTalentFulfillmentJobPost(talentFulfillment));

// Sans payload
dispatch(getAllTalentRequests());                      // ← pas de payload
dispatch(getAllJobPosts());
```

---

## 10. Services Axios (HTTP Client)

### Concept

Les **services** encapsulent les appels HTTP avec Axios. Ils sont appelés par les async thunks.

### Exemple TAMS

```js
// src/features/TalentRequest/talentRequestService.js
import axios from "axios";

const talent_request_service =
  "http://localhost:8080/talent-request-service/talent-request";  // ← URL de base via API Gateway

const createTalentRequest = async (talentRequest) => {
  const response = await axios.post(talent_request_service, talentRequest);  // ← POST
  return response.data;    // ← retourne uniquement le body
};

const getAllTalentRequests = async () => {
  const response = await axios.get(talent_request_service);        // ← GET (liste)
  return response.data;
};

const getTalentRequestById = async (talentRequestId) => {
  const response = await axios.get(talent_request_service + "/" + talentRequestId);  // ← GET (par ID)
  return response.data;
};

const talentRequestService = {
  createTalentRequest,
  getAllTalentRequests,
  getTalentRequestById,
};

export default talentRequestService;
```

### URL pattern

```
Client (port 3000)
  → http://localhost:8080/talent-request-service/talent-request    ← API Gateway (port 8080)
    → discovery (Eureka)                                           ← Service Discovery
      → talent-request-service (port aléatoire)                    ← Microservice
        → Controller REST → Command/Query Gateway                  ← Spring Boot
```

| Service | URL Axios | Route API Gateway |
|---------|-----------|-------------------|
| TalentRequest | `http://localhost:8080/talent-request-service/talent-request` | `talent-request-service/talent-request` |
| TalentFulfillment | `http://localhost:8080/talent-fulfillment-service/talent-fulfillment` | `talent-fulfillment-service/talent-fulfillment` |
| JobPost | `http://localhost:8080/career-portal-service/job-post` | `career-portal-service/job-post` |

---

## 11. Hooks Redux : useSelector, useDispatch

### Concept

| Hook | Rôle |
|------|------|
| `useSelector(fn)` | Lire une partie du state global |
| `useDispatch()` | Obtenir la fonction `dispatch` pour envoyer des actions |

### Exemples dans TAMS

```jsx
import { useSelector, useDispatch } from "react-redux";

// Lire le state
const { talentRequests, isLoading, isSuccess, isError, message } =
  useSelector((state) => state.talentRequests);

const { jobPosts } = useSelector((state) => state.jobPosts);

const { talentFulfillments } = useSelector((state) => state.talentFulfillments);

// Dispatcher une action
const dispatch = useDispatch();
dispatch(getAllTalentRequests());
dispatch(createTalentRequest(talentRequest));
```

### Pattern complet : lecture + dispatch

```jsx
function TalentRequestsView() {
  // 1. Lire le state
  const { talentRequests, isSuccess, isLoading } = useSelector(
    (state) => state.talentRequests
  );

  // 2. Obtenir dispatch
  const dispatch = useDispatch();

  // 3. Charger les données au montage
  useEffect(() => {
    dispatch(getAllTalentRequests());
  }, [dispatch]);

  // 4. Afficher les données
  return (
    <div>
      {talentRequests.map((talentRequest) => (
        <TalentRequestItem key={talentRequest.talentRequestId} talentRequest={talentRequest} />
      ))}
    </div>
  );
}
```

---

## 12. Conditional Rendering

### Concept

Afficher différents éléments UI selon l'état (chargement, erreur, données).

### Patterns dans TAMS

#### Pattern 1 : Loading Spinner

```jsx
function TalentRequestByIdView() {
  const { talentRequest, isError, message, isLoading } = useSelector(...);

  if (isLoading) {
    return <LoadingSpinner />;    // ← écran de chargement
  }

  if (isError) {
    return <h3>Something Went Wrong</h3>;    // ← écran d'erreur
  }

  // ← écran normal avec les données
  return (
    <div className="talent-request-page">
      <h2>{talentRequest.talentRequestTitle}</h2>
      {/* ... */}
    </div>
  );
}
```

#### Pattern 2 : Composant différent selon statut

```jsx
// TalentFulfillmentByIdView.jsx
const displayViewOrFormDependingOnApprovalStatus = (talentFulfillment) => {
  if (talentFulfillment.requestStatus === "APPROVED") {
    return <ApprovedTalentFulfillmentByIdView approvedTalentFulfillment={talentFulfillment} />;
  }
  // Sinon (OPEN ou ASSIGNED_TO_TA)
  return (
    <UpdateAndApproveTalentFulfillmentByIdView
      assignedTalentFulfillment={talentFulfillment}
    />
  );
};

return (
  <div className="talent-request-page">
    {displayViewOrFormDependingOnApprovalStatus(talentFulfillment)}
  </div>
);
```

#### Pattern 3 : Classes CSS dynamiques

```jsx
// Badge de statut avec couleur dynamique
<div className={`status status-${talentRequest.requestStatus}`}>
  {talentRequest.requestStatus}
</div>

// Résultat CSS :
// <div class="status status-OPEN">OPEN</div>          → gris
// <div class="status status-ASSIGNED_TO_TA">...</div> → bleu
// <div class="status status-APPROVED">APPROVED</div>  → vert
```

---

## 13. Gestion des Événements & Formulaires

### Concept

React utilise des événements synthétiques (`onChange`, `onSubmit`, `onClick`) avec des fonctions handler.

### Formulaire complet dans TAMS

```jsx
function CreateTalentRequestForm() {
  // 1. State local du formulaire
  const [talentRequest, setTalentRequest] = useState({
    talentRequestTitle: "",
    jobDescription: { responsibilities: "", qualifications: "" },
    candidateSkills: { coreSkill: "", skillLevel: "" },
    startDate: "",
  });

  // Destructuration pour accès direct
  const { talentRequestTitle, startDate } = talentRequest;

  // 2. Handler onChange (champ simple)
  const onChange = (event) =>
    setTalentRequest({
      ...talentRequest,
      [event.target.name]: event.target.value,   // ← nom du champ = clé dans l'objet
    });

  // 3. Handler onChange (champ imbriqué)
  const onChangeCandidateSkills = (event) =>
    setTalentRequest({
      ...talentRequest,
      candidateSkills: {
        ...talentRequest.candidateSkills,
        [event.target.name]: event.target.value,
      },
    });

  // 4. Handler onSubmit
  const onSubmit = (e) => {
    e.preventDefault();                           // ← empêche le rechargement de la page
    dispatch(createTalentRequest(talentRequest)); // ← dispatch le thunk
  };

  // 5. Rendu du formulaire
  return (
    <form onSubmit={onSubmit}>
      <div className="form-group">
        <label>Title</label>
        <input
          type="text"
          name="talentRequestTitle"             // ← correspond à la clé dans l'objet state
          value={talentRequestTitle}
          onChange={onChange}
        />
      </div>

      <div className="form-group">
        <select
          name="coreSkill"
          value={talentRequest.candidateSkills.coreSkill}
          onChange={onChangeCandidateSkills}
        >
          <option value="">Select Core Skill</option>
          <option value="JAVA">Java</option>
          <option value="REACT">React</option>
          {/* ... */}
        </select>
      </div>

      <button className="btn btn-block" type="submit">
        Submit Talent Request
      </button>
    </form>
  );
}
```

### Types d'événements

| Événement | Usage | Exemple |
|-----------|-------|---------|
| `onSubmit` | Soumission formulaire | `const onSubmit = (e) => { e.preventDefault(); ... }` |
| `onChange` | Mise à jour champ | Mise à jour du state à chaque frappe |
| `onClick` | Clic bouton/lien | Navigation ou action (moins utilisé car géré par Link/Redux) |

---

## 14. Flux de Données Complet

### Diagramme

```
                    ┌─────────────────────────────────────────────────┐
                    │                   Component                     │
                    │  (Affiche UI + Dispatch actions + Select state) │
                    └──────┬──────────────────────────────┬───────────┘
                           │                              │
                    useSelector()                   dispatch()
                           │                              │
                           ▼                              ▼
                    ┌──────────┐               ┌──────────────────────┐
                    │ Redux    │               │  Async Thunk         │
                    │ Store    │◄──────────────│  (createAsyncThunk)  │
                    │ (State)  │               │                      │
                    └──────────┘               │ → pending()          │
                           ▲                   │ → fulfilled(payload) │
                           │                   │ → rejected(error)    │
                           │                   └──────────┬───────────┘
                           │                              │
                           │                              ▼
                           │                    ┌──────────────────────┐
                           │                    │  Axios Service       │
                           │                    │  (HTTP Request)      │
                           │                    └──────────┬───────────┘
                           │                              │
                           │                              ▼
                           │                    ┌──────────────────────┐
                           │                    │  API Gateway :8080   │
                           │                    │  → Backend Axon      │
                           │                    └──────────────────────┘
```

### Flux complet : Création d'une TalentRequest

```
1. User remplit le formulaire (CreateTalentRequestForm)
   → useState gère l'état local du formulaire

2. User clique "Submit"
   → onSubmit(e) {
       e.preventDefault()
       dispatch(createTalentRequest(talentRequest))   ← dispatch async thunk
     }

3. Redux Toolkit :
   → createAsyncThunk s'exécute
   → Dispatch { type: "talentRequests/createTalentRequest/pending" }
   → talentRequestService.createTalentRequest() appelé

4. Axios (talentRequestService.js) :
   → POST http://localhost:8080/talent-request-service/talent-request
   → Attend la réponse du backend

5. Réponse reçue :
   → SUCCÈS : dispatch { type: ".../fulfilled", payload: response.data }
   → ERREUR : dispatch { type: ".../rejected", payload: error.message }

6. Redux Store (talentRequestSlice.js) :
   → pending :    state.isLoading = true
   → fulfilled :  state.talentRequest = action.payload, state.isSuccess = true
   → rejected :   state.isError = true, state.message = action.payload

7. React (via useSelector) détecte le changement d'état :
   → isSuccess = true → navigate("/get-all-talent-requests")
   → isError = true   → toast.error(message)

8. TalentRequestsView s'affiche :
   → useEffect → dispatch(getAllTalentRequests())
   → Redux reçoit la liste
   → talentRequests.map() → affiche chaque item
```

---

## 15. Bonnes Pratiques & Patterns

### Séparation des responsabilités

```
┌─────────────────────────────────────────────────────┐
│  Components/          Pages/                        │
│    BackButton.jsx       TAMSHome.jsx                │
│    Header.jsx          (UI pure, dispatch, select)  │
│    LoadingSpinner.jsx                               │
│    *View.jsx                                        │
│    *Item.jsx                                        │
├─────────────────────────────────────────────────────┤
│  Features/                                          │
│    talentRequestSlice.js    (State + Reducers)      │
│    talentRequestService.js  (API calls)             │
├─────────────────────────────────────────────────────┤
│  App.js                     (Routes + Layout)       │
│  index.js                   (Provider + Bootstrap)  │
│  index.css                  (Styles globaux)        │
└─────────────────────────────────────────────────────┘
```

### Règles d'or

| Règle | Explication |
|-------|-------------|
| **Immutabilité** | Toujours créer un nouvel objet/tableau avec `...spread` ou `action.payload` |
| **État local vs global** | `useState` pour les formulaires, Redux pour les données partagées |
| **Unidirectional data flow** | Les données descendent (props), les actions montent (dispatch) |
| **Components = pure functions** | Même props = même rendu |
| **Side effects dans useEffect** | Appels API, timers, subscriptions |
| **Loading/Error/Success states** | Toujours gérer les 3 états dans l'UI |
| **Destructuration des props** | `function Comp({ prop1, prop2 })` plutôt que `props.prop1` |

### Ressources

| Concept | Documentation |
|---------|--------------|
| React 18 | https://react.dev |
| Redux Toolkit | https://redux-toolkit.js.org |
| React Router 6 | https://reactrouter.com |
| Axios | https://axios-http.com |
| React Toastify | https://fkhadra.github.io/react-toastify |
