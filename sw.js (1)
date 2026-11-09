/**
 * BNR Service Worker
 * Cache management & offline fallback capability
 */

const CACHE_NAME = 'bnr-pwa-v1';
const ASSETS_TO_CACHE = [
    './',
    './index.html',
    './style.css',
    './script.js',
    './manifest.json'
];

self.addEventListener('install', (event) => {
    event.waitUntil(
        caches.open(CACHE_NAME).then((cache) => {
            return cache.addAll(ASSETS_TO_CACHE);
        })
    );
    self.skipWaiting();
});

self.addEventListener('activate', (event) => {
    event.waitUntil(
        caches.keys().then((cacheNames) => {
            return Promise.all(
                cacheNames.map((cache) => {
                    if (cache !== CACHE_NAME) {
                        return caches.delete(cache);
                    }
                })
            );
        })
    );
    self.clients.claim();
});

self.addEventListener('fetch', (event) => {
    // Pass non-GET or Firebase Database requests straight to network
    if (event.request.method !== 'GET' || event.request.url.includes('firebasedatabase.app')) {
        return;
    }

    event.respondWith(
        caches.match(event.request).then((cachedResponse) => {
            if (cachedResponse) {
                // Fetch fresh copy in background
                fetch(event.request).then((networkResponse) => {
                    if (networkResponse.status === 200) {
                        caches.open(CACHE_NAME).then((cache) => cache.put(event.request, networkResponse));
                    }
                }).catch(() => {});
                return cachedResponse;
            }
            return fetch(event.request);
        })
    );
});
