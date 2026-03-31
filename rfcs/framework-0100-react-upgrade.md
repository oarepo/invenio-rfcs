- Start Date: 2026-03-30
- RFC PR: [#<PR>](https://github.com/inveniosoftware/rfcs/pull/<PR>)
- Authors: Miroslav Bauer <bauer@cesnet.cz>

# React v16 to v18 Migration

## Summary

Upgrade all Invenio JavaScript packages from React 16.13.0 (EOL since 2020) to React 18.3,
migrate the test suite from Enzyme to React Testing Library (RTL), optionally adopt TypeScript in the
packages under our control (react-invenio-forms, react-searchkit, possibly Python-based later),
and unify the way JavaScript tests and linting are invoked across all packages.

The migration targets the InvenioRDM v14 release cycle and covers three npm packages (react-overridable,
react-invenio-forms, react-searchkit) plus approx. ten or more Python packages that expose React
entry points (invenio-app-rdm, invenio-rdm-records, invenio-communities, invenio-requests,
invenio-search-ui, invenio-administration, invenio-theme, and others).

## Motivation

React 16 EOL since March 2020, this creates
several compounding problems:
    - Security exposure: npm audit regularly flags vulnerabilities with no upstream fix path.
    - Modern NPM packages increasingly drop support for React 16 (e.g. visualization tools, popular UI/component libraries), some of them were never written with v16 in mind (e.g. AI tools mostly assume v18+ and can
    benefit greatly from proper codebase typing with TS)
    - Contributor friction: Developers who join with current React knowledge (hooks, functional
components, concurrent features) encounter class components, legacy Redux connect() HOCs, and
Enzyme—patterns they have never used professionally. Onboarding takes weeks instead of days, and
potential contributors drop out (no-one wants to learn already obsoleted/legacy concepts).
    - Performance, accessibility, UX/DX improvements - current codebase cannot take advantage of improvements like automatic batching, concurrent features, `useId` hook for accessibility, proper type hints & validation

## User stories this supports

1. **As a new frontend contributor**, I want to write React components and tests using the patterns
    I already know (hooks, RTL, TypeScript), so that I can submit my first PR without studying React-16-specific archaeology.

2. **As a core team developer**, I want code reviews to focus on logic and correctness rather than explaining already deprecated lifecycle methods, so that team velocity increases.

3. **As a module maintainer (Python package with React entry points)**, I want to upgrade my react-invenio-forms peer dependency to React 18 and add modern npm packages without forking any library, so that I can keep security audits clean and use modern visualization tools & integrations in my module.

4. **As an InvenioRDM app developer** used to TypeScript, I want Invenio component libraries to export proper types, so that my team gets IDE autocomplete and compile-time safety without the need to dig through docs & codebase, hunting for scattered ProTypes & defaultProps declarations.

## Expected outcome

All Invenio React packages running on React 18.3.x with modern tooling, including:
- Single way to run lint, tests - every package uses a unified `run-js-tests` / `run-js-lint`
- Adoption of new features (automatic batching, useTransition, useId)
- Enzyme tests replaced by React Testing Library
- Documented migration guides for partners and downstream maintainers
- (Optional) TypeScript adoption (TBD if this should be a part of migration)

## Detailed design

The migration is split into three phases to be aligned to the InvenioRDM v14 release train.
Every package will require major release bump (breaking changes are introduced).

### Phase 1 – Core npm packages

| Package               | Changes                                              |
| --------------------- | ---------------------------------------------------- |
| `eslint-config-invenio` | Automatic JSX transform rules, Enzyme deprecation, other v18 rules |
| `react-overridable`   | React 18.3, RTL, JavaScript kept                                |
| `react-invenio-forms` | React 18.3, Formik 2.4 → 3.x, RTL, TypeScript (optional)        |
| `react-searchkit`     | React 18.3, Redux v8, RTL, TypeScript for public API (optional) |

### Phase 2 – Python packages with React entry points (in parallel)**

`invenio-app-rdm`, `invenio-rdm-records`, `invenio-communities`, `invenio-requests`, `invenio-search-ui`,
`invenio-administration`, `invenio-theme`, and all other packages that declare a `webpack.py` entry point creating a React app.

Changes per Python package:

- Update `webpack.py` / entry-point files to use `ReactDOM.createRoot()` instead of
  `ReactDOM.render()`.
- Update `React`-related dependencies to `react: "^18.0.0"`.
- Migrate any Enzyme tests to RTL.
- Validate build, run e2e tests, using invenio-testrig.

### Phase 3 – Adoption of new features**

If time allows:

#### 1. `useTransition`

Mark non-urgent state updates as "transitions" that can be interrupted by urgent updates (typing, clicking).

**Priority Components**:

**1. Search with Filters** (invenio-search-ui, react-searchkit, invenio-app-rdm)
```javascript
import { startTransition } from 'react';

const handleSearchChange = (value) => {
  // Urgent: Update query immediately
  setSearchQuery(value);
  
  // Transition: results can wait
  startTransition(() => {
    setFilterResults(filterItems(value));
  });
};
```

**2. Record Deposit Forms** (invenio-rdm-records)
```javascript
const handleMetadataChange = (newMetadata) => {
  // Urgent: Update form state
  setFormData(newMetadata);
  
  // Transition: Validation, previews can lag
  startTransition(() => {
    validateForm(newMetadata);
    updatePreview(newMetadata);
  });
};
```

#### 2. `useDeferredValue`  (e.g. for Search Input Throttling) - High Priority

Defer re-rendering non-urgent parts of the tree. Like debouncing but smarter - no fixed delay, interruptible, respects concurrent rendering.

**Priority Components**:

**A. Search Bars** (All search components)
```javascript
import { useDeferredValue } from 'react';

const SearchBar = () => {
  const [query, setQuery] = useState('');
  const deferredQuery = useDeferredValue(query);
  
  // Expensive search results use deferred query
  const results = useSearchResults(deferredQuery);
  
  return (
    <>
      <input 
        value={query} 
        onChange={(e) => setQuery(e.target.value)} 
      />
      <SearchResults results={results} />
    </>
  );
};
```

**B. Large List Filtering** (invenio-communities member lists)
```javascript
const [filterText, setFilterText] = useState('');
const deferredFilter = useDeferredValue(filterText);

const filteredMembers = members.filter(m => 
  m.name.includes(deferredFilter)
);
```

**Benefit**: Input stays responsive while search results lag behind. Better than traditional debouncing because React can interrupt the deferred render if new input arrives.

#### 3. `useId` for Accessibility - Medium Priority

Generate unique IDs that are stable across server/client rendering, avoiding hydration mismatches.

**Priority Components**:

**A. Form Fields** (All deposit forms)
```javascript
import { useId } from 'react';

const FormField = ({ label, error }) => {
  const id = useId();
  const errorId = useId();
  
  return (
    <div>
      <label htmlFor={id}>{label}</label>
      <input 
        id={id} 
        aria-describedby={error ? errorId : undefined} 
      />
      {error && <span id={errorId}>{error}</span>}
    </div>
  );
};
```

**B. Modal Dialogs** (All modal components)
```javascript
const Modal = ({ title, children }) => {
  const titleId = useId();
  const descId = useId();
  
  return (
    <div 
      role="dialog"
      aria-labelledby={titleId}
      aria-describedby={descId}
    >
      <h2 id={titleId}>{title}</h2>
      <div id={descId}>{children}</div>
    </div>
  );
};
```

**Benefit**: Better accessibility for screen readers, no hydration mismatches, replaces hacky ID generation patterns.

#### 4. Suspense for Code Splitting - Medium Priority

Declarative loading states for lazy-loaded components with better integration with transitions.

**Priority Components**:

**A. Landing Page Sections** (invenio-app-rdm)
```javascript
import { lazy, Suspense } from 'react';

const RecordManagement = lazy(() => import('./RecordManagement'));
const CommunitiesManagement = lazy(() => import('./CommunitiesManagement'));
const ExportDropdown = lazy(() => import('./ExportDropdown'));

function LandingPage() {
  return (
    <Suspense fallback={<LoadingSkeleton />}>
      <RecordManagement />
      <Suspense fallback={<SectionLoader />}>
        <CommunitiesManagement />
      </Suspense>
      <ExportDropdown />
    </Suspense>
  );
}
```

**B. Administration Interfaces** (invenio-administration)
```javascript
const DetailsView = lazy(() => import('./DetailsView'));
const EditForm = lazy(() => import('./EditForm'));

function AdminPage() {
  return (
    <Suspense fallback={<AdminLoader />}>
      <DetailsView />
      <EditForm />
    </Suspense>
  );
}
```

**Benefit**: Cleaner loading state management, progressive page loading, better UX with Suspense boundaries.

#### 5. `useSyncExternalStore` (Future - react-redux v8 upgrade)

Consider upgrading react-redux from v7.2.9 to v8+ to leverage React 18's `useSyncExternalStore` hook for better concurrent rendering support. This is handled by the react-redux library internally - no code changes required in InvenioRDM.

#### TypeScript for NPM packages.

- Setup typescript in pure NPM packages, provide typing atleast of public-facing API.

#### Typescript for Python modules

- Configure RSPack build pipeline for Typescript, while still allowing pure JS.
- TypeScript for any new Python entry points (new code only).


## How we teach this

### For Developers

**Official React Documentation**:
- [React.dev](https://react.dev/) - Main React documentation
- [React 18 Upgrade Guide](https://react.dev/blog/2022/03/08/react-18-upgrade-guide)
- [React Hooks Reference](https://react.dev/reference/react)

### For Users

- No user-facing changes expected during core migration
- Performance improvements may be noticeable after Phase 3
- Gradual rollout reduces risk
- Improved responsiveness in search/filter operations

### Documentation Updates

- Update all developer documentation to reference React 18
- Create migration guide / tooling for existing Invenio instances
- Document React 18 feature adoption patterns

## Drawbacks

| Risk | Impact | Likelihood | Mitigation |
|------|--------|------------|------------|
| Ecosystem package delays | High | Medium | Parallel workstreams, early forks |
| Test coverage gaps | High | Medium | Phase 0 focus, no migration without tests |
| Performance regression | Medium | Low | Benchmark tests, monitoring |
| Class component conversion bugs | Medium | Medium | Thorough testing, code review |
| Redux/dependency incompatibility | High | Low | Early compatibility testing |
| Phase 3 feature complexity | Medium | Medium | Incremental adoption, training |

### Known Limitations

1. **ErrorBoundary components**: Must remain class components (React API limitation)

## Alternatives


### Alternative 1: Stay on React 16
**Impact**: 
- ✅ No migration effort required
- ❌ React 16 is several major versions behind (March 2020)
- ❌ Cannot use modern React features
- ❌ Growing ecosystem incompatibility
- ❌ Recruitment challenges (outdated tech stack)

**Recommendation**: Not viable long-term

### Alternative 2: Skip to React 19
**Impact**:
- ✅ Newer version
- ❌ Higher risk (React 19 is newer, less battle-tested)
- ❌ Less ecosystem support
- ❌ Not supported by Semantic-UI React - framework migration required

**Recommendation**: React 18 is safer choice

### Alternative 3: Gradual migration without ecosystem forks
**Impact**:
- ✅ Less upfront work
- ❌ Cannot proceed without ecosystem package updates
- ❌ Dependency conflicts inevitable

### Alternative 5: Migration without Phase 3 (Core migration only)
**Impact**:
- ✅ Shorter timeline (save 3-4 weeks)
- ❌ Miss opportunity for UX improvements
- ❌ No performance optimizations from concurrent features
- ❌ Less value from migration investment

## Unresolved questions

### TBD Items

1. **PropTypes strategy**: Migrate to TypeScript, keep PropTypes, use JSDoc, prop default values?
2. **Optional upgrades**: 
   - react-redux v7.2.9 → v8.x?
   - react-dnd v11.1.3 → v14.x+?
   - uppy v5.x.x (\w React hooks support)? 
3. **CI/CD integration**: How to integrate migration testing into existing pipelines (matrix testing multiple React versions, testrig)
4. **Phase 3 prioritization**: Which new features provide most value?
5. **Class components strategy**: Keep, or (atleast partially) rewrite to functional?

## Resources/Timeline

CESNET is willing to commit resources to help implement this.