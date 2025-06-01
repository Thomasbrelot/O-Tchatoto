<script>
  // La fonction toggleDrawer est passée en prop par le parent (ex: +page.svelte)
  export let toggleDrawer;
</script>

<header class="app-header">
  <button
    class="burger-button"
    type="button"
    on:click={toggleDrawer}
    aria-label="Ouvrir le menu de navigation"
  >
    <span class="bar"></span>
    <span class="bar"></span>
    <span class="bar"></span>
  </button>

<<<<<<< HEAD
	<img class="logo-navbar" src="/image/Logo-O'Tchatoto.png" alt="Logo du site o'tchatoto" />
	<h1 class="app-title">O'Tchatoto</h1>
=======
  <img src="/image/Logo-O'Tchatoto.png" alt="Représentation du logo du tchat 'O'Tchatoto'" />
  <h1 class="app-title">O'Tchatoto</h1>
>>>>>>> sauvegarde-avant-pull
</header>
