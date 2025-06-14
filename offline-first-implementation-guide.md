# Offline-First Implementation Guide

## Overview

This guide provides detailed implementation strategies for building a truly offline-first note-taking application using modern web technologies.

## Core Concepts

### What Makes an App Truly Offline-First?

1. **Local-First Data**: Primary data storage is on the device
2. **Sync as Enhancement**: Network sync improves experience but isn't required
3. **Instant Availability**: Zero loading states for local operations
4. **Conflict Resolution**: Graceful handling of concurrent edits
5. **Progressive Enhancement**: Features degrade gracefully offline

## Technology Stack Deep Dive

### Local Database: RxDB

```typescript
// Initialize RxDB with IndexedDB adapter
import { createRxDatabase } from 'rxdb';
import { getRxStorageDexie } from 'rxdb/plugins/storage-dexie';

const db = await createRxDatabase({
  name: 'notesdb',
  storage: getRxStorageDexie(),
  multiInstance: true, // Sync between tabs
  eventReduce: true   // Optimize performance
});

// Define collections schema
await db.addCollections({
  notes: {
    schema: {
      version: 0,
      primaryKey: 'id',
      type: 'object',
      properties: {
        id: { type: 'string', maxLength: 100 },
        title: { type: 'string' },
        content: { type: 'object' }, // Tiptap JSON
        workspaceId: { type: 'string' },
        parentId: { type: 'string' },
        version: { type: 'number' },
        updatedAt: { type: 'number' },
        deleted: { type: 'boolean' }
      },
      required: ['id', 'workspaceId', 'version']
    }
  }
});
```

### Service Worker Strategy

```javascript
// service-worker.js
const CACHE_NAME = 'notes-app-v1';
const urlsToCache = [
  '/',
  '/static/js/bundle.js',
  '/static/css/main.css'
];

// Install - cache essential files
self.addEventListener('install', event => {
  event.waitUntil(
    caches.open(CACHE_NAME)
      .then(cache => cache.addAll(urlsToCache))
  );
});

// Fetch - network first, fallback to cache
self.addEventListener('fetch', event => {
  if (event.request.url.includes('/api/')) {
    // API calls: network first
    event.respondWith(
      fetch(event.request)
        .then(response => {
          // Clone response for cache
          const responseToCache = response.clone();
          caches.open(CACHE_NAME)
            .then(cache => {
              cache.put(event.request, responseToCache);
            });
          return response;
        })
        .catch(() => {
          // Fallback to cache for API calls
          return caches.match(event.request);
        })
    );
  } else {
    // Static assets: cache first
    event.respondWith(
      caches.match(event.request)
        .then(response => response || fetch(event.request))
    );
  }
});
```

### Sync Engine Architecture

```typescript
// sync-engine.ts
interface SyncEngine {
  push(): Promise<void>;
  pull(): Promise<void>;
  resolveConflicts(conflicts: Conflict[]): Promise<void>;
}

class OfflineFirstSyncEngine implements SyncEngine {
  private db: RxDatabase;
  private api: ApiClient;
  private syncInProgress = false;
  
  async push() {
    if (this.syncInProgress || !navigator.onLine) return;
    
    this.syncInProgress = true;
    
    try {
      // Get all unsynced changes
      const changes = await this.db.notes.find({
        selector: {
          $or: [
            { '_meta.synced': false },
            { '_meta.synced': { $exists: false } }
          ]
        }
      }).exec();
      
      // Batch push to server
      if (changes.length > 0) {
        const response = await this.api.pushChanges({
          changes: changes.map(doc => ({
            id: doc.id,
            data: doc.toJSON(),
            version: doc.version,
            operation: doc.deleted ? 'delete' : 'upsert'
          }))
        });
        
        // Handle conflicts
        if (response.conflicts) {
          await this.resolveConflicts(response.conflicts);
        }
        
        // Mark as synced
        await Promise.all(
          changes.map(doc => 
            doc.atomicPatch({ '_meta.synced': true })
          )
        );
      }
    } finally {
      this.syncInProgress = false;
    }
  }
  
  async pull() {
    if (this.syncInProgress || !navigator.onLine) return;
    
    try {
      const lastSync = await this.getLastSyncTimestamp();
      const updates = await this.api.pullChanges({ since: lastSync });
      
      // Apply remote changes
      for (const update of updates.changes) {
        const existing = await this.db.notes.findOne(update.id).exec();
        
        if (!existing || existing.version < update.version) {
          await this.db.notes.atomicUpsert(update);
        }
      }
      
      await this.setLastSyncTimestamp(updates.timestamp);
    } catch (error) {
      console.error('Pull failed:', error);
    }
  }
  
  async resolveConflicts(conflicts: Conflict[]) {
    for (const conflict of conflicts) {
      // Simple last-write-wins strategy
      const localDoc = await this.db.notes.findOne(conflict.id).exec();
      const remoteDoc = conflict.remoteData;
      
      if (localDoc.updatedAt > remoteDoc.updatedAt) {
        // Keep local version, increment version
        await localDoc.atomicPatch({
          version: remoteDoc.version + 1,
          '_meta.synced': false
        });
      } else {
        // Accept remote version
        await this.db.notes.atomicUpsert(remoteDoc);
      }
    }
  }
}
```

### Optimistic UI Updates

```typescript
// hooks/useOptimisticNote.ts
import { useMutation, useQueryClient } from '@tanstack/react-query';

export function useOptimisticNoteUpdate() {
  const queryClient = useQueryClient();
  
  return useMutation({
    mutationFn: async (update: NoteUpdate) => {
      // 1. Optimistically update local DB
      const localNote = await db.notes.findOne(update.id).exec();
      await localNote.atomicPatch({
        ...update,
        version: localNote.version + 1,
        updatedAt: Date.now(),
        '_meta.synced': false
      });
      
      // 2. Try to sync (non-blocking)
      syncEngine.push().catch(console.error);
      
      return localNote;
    },
    
    onMutate: async (update) => {
      // Cancel in-flight queries
      await queryClient.cancelQueries(['note', update.id]);
      
      // Snapshot previous value
      const previousNote = queryClient.getQueryData(['note', update.id]);
      
      // Optimistically update cache
      queryClient.setQueryData(['note', update.id], old => ({
        ...old,
        ...update
      }));
      
      return { previousNote };
    },
    
    onError: (err, update, context) => {
      // Rollback on error
      queryClient.setQueryData(
        ['note', update.id], 
        context.previousNote
      );
    }
  });
}
```

### Editor Integration with Offline Storage

```typescript
// components/Editor.tsx
import { useEditor } from '@tiptap/react';
import StarterKit from '@tiptap/starter-kit';
import { useEffect, useCallback } from 'react';
import { debounce } from 'lodash';

export function OfflineFirstEditor({ noteId }) {
  const [note, setNote] = useState(null);
  const [syncStatus, setSyncStatus] = useState('synced');
  
  // Load from local DB
  useEffect(() => {
    const subscription = db.notes
      .findOne(noteId)
      .$.subscribe(doc => {
        setNote(doc);
        setSyncStatus(doc?._meta?.synced ? 'synced' : 'pending');
      });
      
    return () => subscription.unsubscribe();
  }, [noteId]);
  
  const editor = useEditor({
    extensions: [StarterKit],
    content: note?.content || '',
    onUpdate: ({ editor }) => {
      saveToLocal(editor.getJSON());
    }
  });
  
  // Debounced save to local DB
  const saveToLocal = useCallback(
    debounce(async (content) => {
      const doc = await db.notes.findOne(noteId).exec();
      await doc.atomicPatch({
        content,
        updatedAt: Date.now(),
        '_meta.synced': false
      });
      
      // Trigger background sync
      syncEngine.push().catch(console.error);
    }, 500),
    [noteId]
  );
  
  return (
    <div>
      <SyncIndicator status={syncStatus} />
      <EditorContent editor={editor} />
    </div>
  );
}
```

## Handling Edge Cases

### 1. Storage Quota Management

```typescript
// storage-manager.ts
class StorageManager {
  async checkQuota() {
    if ('storage' in navigator && 'estimate' in navigator.storage) {
      const estimate = await navigator.storage.estimate();
      const percentUsed = (estimate.usage / estimate.quota) * 100;
      
      if (percentUsed > 90) {
        await this.cleanupOldData();
      }
      
      return {
        usage: estimate.usage,
        quota: estimate.quota,
        percentUsed
      };
    }
  }
  
  async cleanupOldData() {
    // Delete old cached attachments
    const attachments = await db.attachments
      .find({
        selector: {
          lastAccessed: {
            $lt: Date.now() - 30 * 24 * 60 * 60 * 1000 // 30 days
          }
        }
      })
      .exec();
      
    await Promise.all(
      attachments.map(att => att.remove())
    );
  }
}
```

### 2. Network State Management

```typescript
// hooks/useNetworkState.ts
export function useNetworkState() {
  const [isOnline, setIsOnline] = useState(navigator.onLine);
  const [connectionType, setConnectionType] = useState(
    navigator.connection?.effectiveType || 'unknown'
  );
  
  useEffect(() => {
    const handleOnline = () => {
      setIsOnline(true);
      // Trigger sync when coming online
      syncEngine.sync();
    };
    
    const handleOffline = () => {
      setIsOnline(false);
    };
    
    const handleConnectionChange = () => {
      setConnectionType(navigator.connection?.effectiveType || 'unknown');
    };
    
    window.addEventListener('online', handleOnline);
    window.addEventListener('offline', handleOffline);
    navigator.connection?.addEventListener('change', handleConnectionChange);
    
    return () => {
      window.removeEventListener('online', handleOnline);
      window.removeEventListener('offline', handleOffline);
      navigator.connection?.removeEventListener('change', handleConnectionChange);
    };
  }, []);
  
  return { isOnline, connectionType };
}
```

### 3. Conflict Resolution UI

```typescript
// components/ConflictResolver.tsx
export function ConflictResolver({ conflict }) {
  const [resolution, setResolution] = useState(null);
  
  const handleResolve = async (choice: 'local' | 'remote' | 'merge') => {
    switch (choice) {
      case 'local':
        await conflict.keepLocal();
        break;
      case 'remote':
        await conflict.keepRemote();
        break;
      case 'merge':
        const merged = await conflict.merge();
        // Show merge UI
        break;
    }
  };
  
  return (
    <Dialog>
      <h2>Sync Conflict Detected</h2>
      <div className="conflict-preview">
        <div className="local-version">
          <h3>Your Version</h3>
          <NotePreview content={conflict.localContent} />
          <time>{formatTime(conflict.localUpdated)}</time>
        </div>
        <div className="remote-version">
          <h3>Server Version</h3>
          <NotePreview content={conflict.remoteContent} />
          <time>{formatTime(conflict.remoteUpdated)}</time>
        </div>
      </div>
      <div className="actions">
        <Button onClick={() => handleResolve('local')}>
          Keep Mine
        </Button>
        <Button onClick={() => handleResolve('remote')}>
          Use Theirs
        </Button>
        <Button onClick={() => handleResolve('merge')}>
          Merge Both
        </Button>
      </div>
    </Dialog>
  );
}
```

## Performance Optimizations

### 1. Lazy Loading Collections

```typescript
// Only load notes in current workspace
const currentWorkspace = await db.workspaces
  .findOne(workspaceId)
  .exec();

const notes = await db.notes
  .find({
    selector: {
      workspaceId: currentWorkspace.id
    },
    limit: 50,
    sort: [{ updatedAt: 'desc' }]
  })
  .exec();
```

### 2. Virtual Scrolling for Large Lists

```typescript
import { FixedSizeList } from 'react-window';

export function NotesList({ notes }) {
  const Row = ({ index, style }) => (
    <div style={style}>
      <NoteItem note={notes[index]} />
    </div>
  );
  
  return (
    <FixedSizeList
      height={600}
      itemCount={notes.length}
      itemSize={80}
      width="100%"
    >
      {Row}
    </FixedSizeList>
  );
}
```

### 3. Web Workers for Heavy Operations

```typescript
// search.worker.ts
import Fuse from 'fuse.js';

let fuse;

self.addEventListener('message', async (e) => {
  const { type, payload } = e.data;
  
  switch (type) {
    case 'INIT_SEARCH':
      const notes = payload.notes;
      fuse = new Fuse(notes, {
        keys: ['title', 'content.text'],
        threshold: 0.3
      });
      break;
      
    case 'SEARCH':
      const results = fuse.search(payload.query);
      self.postMessage({
        type: 'SEARCH_RESULTS',
        results: results.slice(0, 50)
      });
      break;
  }
});
```

## Testing Offline Scenarios

```typescript
// tests/offline.test.ts
describe('Offline functionality', () => {
  beforeEach(() => {
    // Simulate offline
    cy.intercept('**/api/**', { forceNetworkError: true });
  });
  
  it('should create and edit notes offline', () => {
    cy.visit('/');
    cy.get('[data-test="new-note"]').click();
    cy.get('[data-test="editor"]').type('Offline note content');
    
    // Should save without errors
    cy.get('[data-test="sync-status"]').should('contain', 'Saved locally');
    
    // Should persist after reload
    cy.reload();
    cy.get('[data-test="editor"]').should('contain', 'Offline note content');
  });
  
  it('should sync when coming online', () => {
    // Create note offline
    cy.get('[data-test="new-note"]').click();
    cy.get('[data-test="editor"]').type('Will sync');
    
    // Come online
    cy.intercept('**/api/**').as('sync');
    
    // Should auto-sync
    cy.wait('@sync');
    cy.get('[data-test="sync-status"]').should('contain', 'Synced');
  });
});
```

## Best Practices

1. **Always Design API for Offline**: Every API endpoint should support partial data and conflict resolution
2. **Progressive Enhancement**: Start with offline functionality, add online features
3. **Clear Sync Status**: Users should always know if their data is synced
4. **Graceful Degradation**: Features that require network should disable cleanly
5. **Test Offline First**: Test offline scenarios before online ones
6. **Handle Edge Cases**: Storage limits, corrupted data, version mismatches
7. **Optimize for Performance**: Use indexes, virtual scrolling, lazy loading

## Conclusion

Building truly offline-first applications requires rethinking traditional web architectures. By putting local storage first and treating network sync as an enhancement, we can create applications that are faster, more reliable, and respect user autonomy over their data.