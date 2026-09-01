// Service worker mínimo — só existe pra deixar a ferramenta instalável como
// app no Android e abrir rápido mesmo com sinal fraco. Não guarda em cache
// nenhum dado da Prefeitura, do mapa ou de endereço: essas buscas sempre vão
// direto pra internet, pra nunca mostrar uma informação desatualizada.

const CACHE_NAME = "quadra-lote-shell-v3";
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
  //
  // IMPORTANTE: quando alguém abre o link "de raiz" (o endereço do site sem
  // terminar em "index.html" — que é como o atalho da tela inicial e a
  // maioria dos acessos realmente abrem), esse pedido tem "mode: navigate"
  // e o caminho da URL não termina em nenhum dos SHELL_FILES — então, sem
  // esse "||" abaixo, ele passava batido pela regra de sempre buscar a
  // versão mais nova, e podia acabar servido do cache comum do navegador
  // (desatualizado) em vez do nosso "sempre busca de novo".
  const ehArquivoDoApp = url.origin === self.location.origin &&
    (event.request.mode === "navigate" ||
      SHELL_FILES.some((f) => url.pathname.endsWith(f.replace("./", ""))));

  if (!ehArquivoDoApp) return;

  // "no-store" força ignorar o cache HTTP do próprio navegador — sem isso,
  // o Chrome às vezes reaproveita uma resposta antiga e a atualização
  // publicada no GitHub Pages não aparece nem no app instalado nem numa
  // aba comum, mesmo com o service worker fazendo tudo certo.
  event.respondWith(
    fetch(event.request, { cache: "no-store" })
      .then((resp) => {
        const copia = resp.clone();
        caches.open(CACHE_NAME).then((cache) => cache.put(event.request, copia));
        return resp;
      })
      .catch(() => caches.match(event.request))
  );
});
