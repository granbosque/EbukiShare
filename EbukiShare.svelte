<script>
  import { onMount } from 'svelte';
  import { BskyAgent } from '@atproto/api';
  import { json } from '@sveltejs/kit';
  
  // Props del componente

  export let libro;
  /* objeto con las siguientes propiedades:
  {
    title: string;
    author: string;
    coverImage: string; // url de la imagen de la portada, se usa para generar la vista previa del enlace en el post (OpenGraph)
    description: string; // descripción del libro, se usa para generar la vista previa del enlace en el post (OpenGraph)
    description2: string; // descripción adicional del libro. Se muestra más pequeña y no se incluye en la vista previa de opengraph
    bsky: string; // handle de bluesky del autor, para mencionarlo en el post

    // formatos de descarga, opcionales, al menos un formato es obligatorio
    file_PDF: string; // url del fichero PDF
    file_PDFsmall: string; // url del fichero PDF tamaño móvil
    file_EPUB: string; // url del fichero EPUB
    file_MOBI: string; // url del fichero MOBI
  }
  */

  // para el resto de props se propiorcionan valores por defecto
  export let platformName = 'ebuki';
  export let platformHandle = '@ebuki.es';

  // Props para colores (solo los necesarios)
  export let primaryColor = '#10b981';
  export let secondaryTextColor = '#4b5563';

  // Props para funciones de estadísticas (opcionales)
  export let onShare = null; // Función que se llamará cuando se comparta un libro
  export let onDownload = null; // Función que se llamará cuando se descargue un libro

  // Extraer propiedades del libro
  export let title = libro.title;
  export let author = libro.author;
  export let bsky = libro.bsky;
  export let imageUrl = libro.coverImage;
  export let bookId = libro.id;


  // si algún formato no está en esta lista, se mostrará el nombre del formato (ej PDF, EPUB)
  const formatDescriptions = {
    PDFsmall: "PDF móvil",
  }; 

  // Función para validar que tenemos todos los datos necesarios
  $: isValidBook = title && author && availableFormats.length > 0;

  // Estado del componente
  let agent = new BskyAgent({ service: 'https://bsky.social' });
  let isLoggedIn = false;
  let username = "";
  let password = "";
  let loginError = "";
  let isLoading = false;
  let showPostEditor = false;
  let defaultPostMessage = `Me estoy descargando "${title}" de ${author}. ¡Recomendado! 📚`;
  let userEditableContent = "";
  let postContent = "";
  let postStatus = "";
  let isDownloadReady = false;
  let userHandle = "";
  let isAuthChecked = false;
  let isCheckingAuth = true; // Nueva variable para controlar el estado de carga inicial

  // Comprobar si el libro ya ha sido "pagado"
  let hasAlreadyPaid = false;

  // Añadir estas variables de estado
  let currentUrl = "";
  let remainingChars = 300;

  onMount(() => {
    // Iniciar verificación de autenticación
    isCheckingAuth = true;
    
    // Comprobar si hay sesión guardada
    const session = localStorage.getItem('bsky_session');
    if (session) {
      try {
        const sessionData = JSON.parse(session);
        if (sessionData && sessionData.exp > Date.now() / 1000) {
          // Restaurar sesión
          agent.resumeSession(sessionData)
            .then(() => {
              isLoggedIn = true;
              userHandle = sessionData.handle;
              isAuthChecked = true;
              isCheckingAuth = false;
            })
            .catch(error => {
              console.error("Error al restaurar sesión:", error);
              // Si hay error al restaurar la sesión, limpiamos el localStorage
              localStorage.removeItem('bsky_session');
              isLoggedIn = false;
              userHandle = "";
              isAuthChecked = true;
              isCheckingAuth = false;
            });
        } else {
          // La sesión ha expirado
          localStorage.removeItem('bsky_session');
          isLoggedIn = false;
          isAuthChecked = true;
          isCheckingAuth = false;
        }
      } catch (e) {
        console.error("Error al procesar la sesión guardada:", e);
        localStorage.removeItem('bsky_session');
        isLoggedIn = false;
        isAuthChecked = true;
        isCheckingAuth = false;
      }
    } else {
      // No hay sesión guardada
      isAuthChecked = true;
      isCheckingAuth = false;
    }

    // Comprobar si el libro ya ha sido compartido anteriormente
    hasAlreadyPaid = localStorage.getItem(`downloaded_${title}`) === 'true';
    if (hasAlreadyPaid) {
      isDownloadReady = true;
    }
    
    // Obtener la URL actual
    currentUrl = window.location.href;
  });

  // Función para calcular caracteres restantes
  // TOD: revisar si funciona bien con caracteres especiales y emojis
  function calculateRemainingChars(text) {
    let fixedContent = '';
    if (bsky) {
      fixedContent += ` @${bsky}`;
    }
    fixedContent += ` (vía @${platformHandle})`;
    fixedContent += ` ${currentUrl}`;  
    return 300 - (text.length + fixedContent.length);
  }

  function startPostProcess() {
    userEditableContent = defaultPostMessage;
    showPostEditor = true;
    isDownloadReady = false;
  }

  async function handleLogin() {
    isLoading = true;
    loginError = "";
    
    try {
      const response = await agent.login({
        identifier: username,
        password: password
      });
      
      isLoggedIn = true;
      userHandle = response.data.handle;
      
      // Guardar sesión solamente en el almacenamiento local
      localStorage.setItem('bsky_session', JSON.stringify({
        ...response.data,
        exp: response.data.refreshJwt ? JSON.parse(atob(response.data.refreshJwt.split('.')[1])).exp : (Math.floor(Date.now() / 1000) + 24 * 60 * 60)
      }));
      
      password = ""; // Limpiar contraseña
    } catch (error) {
      loginError = "Error al iniciar sesión. Verifica tus credenciales.";
      console.error("Login error:", error);
    } finally {
      isLoading = false;
    }
  }

  function handleLogout() {
    agent = new BskyAgent({ service: 'https://bsky.social' });
    isLoggedIn = false;
    localStorage.removeItem('bsky_session');
    userHandle = "";
    showPostEditor = false;
    isDownloadReady = hasAlreadyPaid;
  }

  async function publishPost() {
    if (remainingChars < 0) {
      postStatus = "Error: El mensaje es demasiado largo. Por favor, acórtalo.";
      return;
    }
    isLoading = true;
    postStatus = "";
    
    try {
      // Crear el post base con las menciones
      // (las menciones hay que especificarlas como facets y embeds, la api de bluesky no las añade automáticamente desde el texto plano)
      const post = {
        text: postContent,
        facets: []
      };

      // Función auxiliar para calcular el índice de bytes
      function getByteLength(str) {
        return new TextEncoder().encode(str).length;
      }

      // Se añade la URL como enlace usando facets
      const urlIndex = post.text.indexOf(currentUrl);
      if (urlIndex >= 0) {
        const byteStart = getByteLength(post.text.slice(0, urlIndex));
        const byteEnd = byteStart + getByteLength(currentUrl);
        
        post.facets.push({
          index: {
            byteStart,
            byteEnd
          },
          features: [{
            $type: "app.bsky.richtext.facet#link",
            uri: currentUrl
          }]
        });
      }

      // Se añade card preview como embed
      if (currentUrl) {
        // Preparar el objeto embed sin la imagen primero
        const embedObj = {
          $type: "app.bsky.embed.external",
          external: {
            uri: currentUrl,
            title: title,
            description: `${title} (${author}) - Vía ${platformName}`
          }
        };
        
        // Si hay una URL de imagen, se descarga, se convierte a blob y se sube a bsky
        if (imageUrl) {
          try {
            // Descargar la imagen
            const imgResponse = await fetch(imageUrl);
            if (imgResponse.ok) {
              const imgBlob = await imgResponse.blob();
              
              // Subir la imagen como blob a Bluesky
              const uploadResult = await agent.uploadBlob(imgBlob, {
                encoding: imgBlob.type
              });
              
              // Añadir la referencia del blob a la card
              if (uploadResult && uploadResult.data && uploadResult.data.blob) {
                embedObj.external.thumb = uploadResult.data.blob;
              }
            }
          } catch (imgError) {
            console.error("Error al cargar la imagen:", imgError);
            // Continuamos sin la imagen si hay error
          }
        }
        
        // Asignar el objeto embed al post
        post.embed = embedObj;
      }

      // Agregar la mención a la plataforma
      if (platformHandle) {
        const platformMention = `@${platformHandle}`;
        const mentionIndex = post.text.indexOf(platformMention);
        if (mentionIndex >= 0) {
          try {
            const platformProfile = await agent.resolveHandle({ handle: platformHandle });
            
            const byteStart = getByteLength(post.text.slice(0, mentionIndex));
            const byteEnd = byteStart + getByteLength(platformMention);

            post.facets.push({
              index: {
                byteStart,
                byteEnd
              },
              features: [{
                $type: "app.bsky.richtext.facet#mention",
                did: platformProfile.data.did
              }]
            });
          } catch (error) {
            console.error("Error resolving platform handle:", error);
          }
        }
      }

      // mención al autor con su handle de Bluesky
      if (bsky) {
        const bskyMention = `@${bsky}`;
        const mentionIndex = post.text.indexOf(bskyMention);
        if (mentionIndex >= 0) {
          try {
            // Primero resolvemos el handle para obtener el DID
            const profile = await agent.resolveHandle({ handle: bsky });
            
            const byteStart = getByteLength(post.text.slice(0, mentionIndex));
            const byteEnd = byteStart + getByteLength(bskyMention);

            post.facets.push({
              index: {
                byteStart,
                byteEnd
              },
              features: [{
                $type: "app.bsky.richtext.facet#mention",
                did: profile.data.did
              }]
            });
          } catch (error) {
            console.error("Error resolving handle:", error);
            // Si falla la resolución del handle, publicamos sin la mención
          }
        }
      }

      // Publicar el post
      const response = await agent.post(post);
      
      // Llamar a la función de estadísticas si existe
      if (onShare) {
        onShare(userHandle, bookId);
      }
      
      // Marcar como pagado y habilitar descarga
      localStorage.setItem(`downloaded_${title}`, 'true');
      isDownloadReady = true;
      showPostEditor = false;
      hasAlreadyPaid = true;
      
    } catch (error) {
      postStatus = "Error al publicar. Inténtalo de nuevo.";
      console.error("Post error:", error);
    } finally {
      isLoading = false;
    }
  }

  async function escribirLog(userHandle, bookId) {
    try {
        await fetch('/.netlify/functions/logShare', {
          method: 'POST',
          headers: {
            'Content-Type': 'application/json'
          },
          body: JSON.stringify({
            userHandle, bookId
          })
        });
      } catch (logError) {
        console.error("Error al registrar compartido en Supabase:", logError);
        // No interrumpimos el flujo principal si esto falla
      }
  }

  async function escribirLogDescarga(userHandle, bookId, format) {
    try {
      await fetch('/.netlify/functions/logDownload', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json'
        },
        body: JSON.stringify({
          userHandle, 
          bookId,
          format
        })
      });
    } catch (logError) {
      console.error("Error al registrar descarga en Supabase:", logError);
      // No interrumpimos el flujo principal si esto falla
    }
  }

  // Función para obtener los formatos disponibles del libro
  function getAvailableFormats(libro) {
    return Object.keys(libro)
      .filter(key => key.startsWith('file_'))
      .map(key => ({
        format: key.replace('file_', ''),
        url: libro[key]
      }));
  }
  
  // Obtener los formatos disponibles
  $: availableFormats = getAvailableFormats(libro);

  // Función para manejar la descarga
  function handleDownload(format, url) {
    // Llamar a la función de estadísticas si existe
    if (onDownload) {
      onDownload(userHandle, bookId, format);
    }
  }
</script>

<style>
  /* Contenedor principal */
  .container {
    --primary-color: #10b981;
    --secondary-text-color: #4b5563;
    --error-color: #e74c3c;
    --error-bg-color: #fde8e8;
    --info-bg-color: #f9fafb;
    --border-radius: 12px;
    --box-shadow: 0 8px 30px rgba(0, 0, 0, 0.12);
    --transition: all 0.3s ease;
  }

  /* Estructura básica */
  section {
    margin: 20px 0;
  }

  /* Elementos de formulario */
  input, textarea {
    width: 100%;
    padding: 12px 16px;
    margin: 8px 0 20px;
    border: 2px solid var(--primary-color);
    border-radius: var(--border-radius);
    background-color: white;
    box-sizing: border-box;
    resize: vertical;
  }

  input:focus, textarea:focus {
    border-color: var(--primary-color);
    outline: none;
    box-shadow: 0 0 0 3px rgba(16, 185, 129, 0.2);
  }

  label {
    font-weight: 600;
    display: block;
    margin-bottom: 5px;
    color: var(--secondary-text-color);
  }

  /* Botones */
  button, .button {
    background-color: var(--primary-color);
    color: white;
    border: none;
    padding: 12px 24px;
    border-radius: var(--border-radius);
    cursor: pointer;
    margin: 10px 0;
    text-decoration: none;
    display: inline-block;
    font-weight: 600;
    transition: var(--transition);
    text-align: center;
  }

  button:hover, .button:hover {
    filter: brightness(1.1);
    transform: translateY(-2px);
  }

  button:active, .button:active {
    transform: translateY(1px);
  }

  button:disabled {
    opacity: 0.6;
    cursor: not-allowed;
    transform: none;
  }

  .button-secondary {
    background: none;
    color: black;
  }

  .button-secondary:hover {
    background-color: #e5e7eb;
  }

  .button-link {
    background: none;
    color: var(--primary-color);
    padding: 0;
    margin: 0;
    font-weight: 500;
  }

  .button-link:hover {
    background: none;
    transform: none;
    text-decoration: underline;
  }

  /* Mensajes y alertas */
  .error {
    color: var(--error-color);
    background-color: var(--error-bg-color);
    padding: 16px;
    border-radius: var(--border-radius);
    margin: 16px 0;
    border-left: 4px solid var(--error-color);
  }

  /* Cajas informativas */
  .info-box {
    text-align: center;
    margin-bottom: 24px;
  }

  .small-text {
    font-size: 0.7rem;
    color: var(--secondary-text-color);
  }

  .steps-box {
    background-color: var(--info-bg-color);
    padding: 16px;
    border-radius: var(--border-radius);
    margin: 16px 0;
    border-left: 4px solid var(--primary-color);
  }

  .steps-box p {
    margin: 8px 0;
    display: flex;
    align-items: center;
  }

  .steps-box p::before {
    content: "";
    display: inline-block;
    width: 24px;
    height: 24px;
    background-color: var(--primary-color);
    border-radius: 50%;
    margin-right: 12px;
    color: white;
    text-align: center;
    line-height: 24px;
    font-weight: bold;
  }

  .steps-box p:nth-child(1)::before {
    content: "1";
  }

  .steps-box p:nth-child(2)::before {
    content: "2";
  }

  .steps-box p:nth-child(3)::before {
    content: "3";
  }

  .info-card {
    background-color: var(--info-bg-color);
    padding: 20px;
    border-radius: var(--border-radius);
    margin: 20px 0;
  }

  .info-item {
    display: flex;
    gap: 16px;
    margin-bottom: 16px;
    align-items: flex-start;
  }

  .info-item i {
    font-size: 1.5rem;
    margin-top: 3px;
  }

  .info-item strong {
    display: block;
    margin-bottom: 5px;
    color: var(--secondary-text-color);
  }

  /* Usuario */
  .user-info {
    background-color: white;
    padding: 20px;
    border-radius: var(--border-radius);
    margin-bottom: 20px;
    padding: 12px 16px;
    display: flex;
    justify-content: space-between;
    align-items: center;
  }
  .user-info, .user-info button {
    font-size: 0.8rem;
  }

  .user-info span {
    font-weight: 600;
    color: var(--secondary-text-color);
  }

  /* Acciones */
  .action-buttons {
    display: flex;
    gap: 12px;
    justify-content: center;
    margin-top: 16px;
  }

  /* Colores dinámicos */
  .primary-text {
    color: var(--primary-color);
    margin-bottom: 0.5em;
    font-weight: 600;
  }

  .primary-icon {
    color: var(--primary-color);
  }

  .download-section {
    max-width: 30em;
    margin: 0 auto;
  }

  .section-subtitle {
    text-align: center;
    margin-bottom: 16px;
    color: var(--secondary-text-color);
    opacity: 0.8;
  }

  .download-button {
    display: block;
    width: 100%;
    max-width: 250px;
    margin: 24px auto;
    padding: 16px 24px;
    font-size: 1.1rem;
  }

  .post-editor {
    text-align: left;
    margin-top: 20px;
  }

  p { 
    margin: 0;
    padding: 0;
    line-height: 1.6;
  }

  .thank-you-message {
    margin: 16px 0;
    font-weight: 500;
    color: var(--primary-color);
  }

  .already-shared {
    margin: 16px 0;
    padding: 12px;
  }

  .char-counter {
    font-size: 0.9em;
    text-align: right;
    color: var(--secondary-text-color);
    margin-top: -16px;
    margin-bottom: 16px;
  }

  .char-counter.error {
    color: var(--error-color);
  }

  .error-input {
    border-color: var(--error-color) !important;
  }

  .loading-indicator {
    text-align: center;
    padding: 20px;
    color: var(--secondary-text-color);
    opacity: 0;
    animation: fadeIn 0.3s ease forwards;
    animation-delay: 1s;
  }

  @keyframes fadeIn {
    from { opacity: 0; }
    to { opacity: 1; }
  }
</style>

<div class="container" style="--primary-color: {primaryColor}; --secondary-text-color: {secondaryTextColor};">

  {#if !isValidBook}
    <div class="error">
      <p><strong>Datos del libro incompletos</strong></p>
      <p>No se han proporcionado todos los datos necesarios para compartir el libro.</p>
    </div>
  {:else}
    <!-- Sección de descarga -->
    <section class="download-section">
      {#if isCheckingAuth}
        <div class="loading-indicator">
          <p>Conectando con bluesky...</p>
        </div>
      {:else if isAuthChecked}
        {#if !isLoggedIn}
          <div class="info-box">
            <h3 class="section-subtitle">Descarga gratuita por compartir</h3>
            
            <div class="steps-box">
              <p>Inicia sesión en tu cuenta de Bluesky</p>
              <p>Comparte un mensaje sobre el libro</p>
              <p>¡Descarga y disfruta!</p>
            </div>
          </div>

          <div>
            <label for="username">
              Tu usuario de Bluesky
            </label>
            <input 
              type="text" 
              id="username" 
              bind:value={username} 
              placeholder="usuario.bsky.social"
            />
          </div>
          
          <div>
            <label for="password">
              Contraseña o App Password
            </label>
            <input 
              type="password" 
              id="password" 
              bind:value={password} 
              placeholder="••••••••"
            />
          </div>
          
          {#if loginError}
            <p class="error">{loginError}</p>
          {/if}
          
          <button 
            on:click={handleLogin} 
            disabled={isLoading || !username || !password}
            class="download-button"
          >
            {isLoading ? 'Iniciando sesión...' : 'Iniciar sesión'}
          </button>

          <div class="info-card">
            <div class="info-item">
              <i class="ri-lock-fill primary-icon"></i>
              <div>
                <p><strong>Tu privacidad es importante</strong></p>
                <p>Este sistema de identificación funciona directamente entre tu navegador y Bluesky. Tus credenciales no se almacenan ni se envían a {platformName} ni a ningún servidor externo.</p>
              </div>
            </div>
            <div class="info-item">
              <i class="ri-flashlight-fill primary-icon"></i>
              <div>
                <p><strong>Consejo de seguridad</strong></p>
                <p>
                  Para mayor seguridad, te recomendamos usar un <a 
                    href="https://bsky.app/settings/app-passwords" 
                    target="_blank" 
                    rel="noopener noreferrer" 
                  >App Password</a> en lugar de tu contraseña principal. Los App Passwords son códigos temporales que puedes revocar en cualquier momento.
                </p>
              </div>
            </div>
          </div>
        {:else if isDownloadReady}
          <div class="info-box">
            <div class="user-info ">
              <span>@{userHandle}</span>
              <button on:click={handleLogout} class="button-link">
                Cerrar sesión
              </button>
            </div>
            
            {#each availableFormats as {format, url}}
              <a 
                href={url} 
                target="_blank" 
                rel="noopener noreferrer"
                class="button download-button"
                on:click={() => handleDownload(format, url)}
              >
                Descargar {formatDescriptions[format] || format}
              </a>
            {/each}
            {#if hasAlreadyPaid && !showPostEditor}
              <div class="already-shared">
                <p>Ya has compartido este libro anteriormente</p>
                <button 
                  on:click={startPostProcess}
                  class="button-link primary-text"
                >
                  Volver a compartir
                </button>
              </div>
            {:else}
              <p class="thank-you-message">¡Gracias por compartir! Disfruta tu lectura</p>
            {/if}
          </div>
        {:else}
          <div class="info-box">
            <div class="user-info">
              <span>@{userHandle}</span>
              <button on:click={handleLogout} class="button-link">
                Cerrar sesión
              </button>
            </div>

            {#if !showPostEditor}
              <button 
                on:click={startPostProcess}
                class="download-button"
              >
                Compartir para descargar
              </button>
              <div class="small-text">Ayuda a difundir este libro</div>
              
            {/if}

            {#if showPostEditor}
              <div class="post-editor">
                <p>Comparte un mensaje sobre el libro en tu cuenta de Bluesky para descargar el ebook. Con este gesto ayudas a que más gente pueda disfrutar de él.</p>

                <textarea 
                  bind:value={userEditableContent}
                  rows="4"
                  class:error-input={remainingChars < 0}
                ></textarea>

                <div class="char-counter" class:error={remainingChars < 0}>
                  {remainingChars} caracteres restantes
                </div>

                <div class="small-text">Se añadirá automáticamete una mención al autor y el enlace de descarga.</div>
                
                {#if postStatus}
                  <p class="error">{postStatus}</p>
                {/if}
                
                <div class="action-buttons">
                  <button 
                    on:click={() => showPostEditor = false}
                    class="button-secondary"
                  >
                    Cancelar
                  </button>
                  <button 
                    on:click={publishPost}
                    disabled={isLoading || !userEditableContent.trim()}
                  >
                    {isLoading ? 'Publicando...' : 'Publicar'}
                  </button>
                </div>
              </div>
            {/if}
          </div>
        {/if}
      {/if}
    </section>

  {/if}
</div>