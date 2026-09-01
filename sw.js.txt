// Service worker mínimo — só existe pra deixar a ferramenta instalável como
// app no Android e abrir rápido mesmo com sinal fraco. Não guarda em cache
// nenhum dado da Prefeitura, do mapa ou de endereço: essas buscas sempre vão
// direto pra internet, pra nunca mostrar uma informação desatualizada.

const CACHE_NAME = "quadra-lote-shell-v1";
const SHELL_FILES = ["./index.html", "./manifest.json", "./icon-192.png", "./icon-512.png"];

self.addEventListener("install", (event) => {
  event.waitUntil(
    caches.open(CACHE_NAME).then((cache) => cache.addAll(SHELL_FILES))
  );
  self.skipWaiting();
});

self.addEventListener("activate", (event) => {
  event.waitUntil(
    caches.keys().then((names) =>
      Promise.all(names.filter((n) => n !== CACHE_NAME).map((n) => caches.delete(n)))
    )
  );
  self.clients.claim();
});

self.addEventListener("fetch", (event) => {
  const url = new URL(event.request.url);

  // só cuida do "casco" do app (o próprio HTML/manifest/ícones, mesma
  // origem). Qualquer outra coisa (dados da Prefeitura, mapas, fontes,
  // bibliotecas) passa direto pra rede, sem passar pelo cache.
  const ehArquivoDoApp = url.origin === self.location.origin &&
    SHELL_FILES.some((f) => url.pathname.endsWith(f.replace("./", "")));

  if (!ehArquivoDoApp) return;

  event.respondWith(
    fetch(event.request)
      .then((resp) => {
        const copia = resp.clone();
        caches.open(CACHE_NAME).then((cache) => cache.put(event.request, copia));
        return resp;
      })
      .catch(() => caches.match(event.request))
  );
});
