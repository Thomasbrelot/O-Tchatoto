<script>
	// Props venant du parent
	export let userMessage;
	export let sendMessage;

	// Fonction appelée lors de l'envoi du formulaire
	function handleSubmit(event) {
		event.preventDefault(); // Empêche le rechargement
		if (!userMessage.trim()) return;
		sendMessage();
	}
</script>

<footer class="chat-footer">
	<!-- Formulaire de chat -->
	<form class="chat-form" on:submit={handleSubmit}>
		<input
			type="text"
			placeholder="Écrire à notre chatoto..."
			class="chat-input"
			bind:value={userMessage}
		/>
		<button type="submit" class="send-button">Envoyer</button>
	</form>

	<!-- Actions supplémentaires -->
	<div class="chat-actions">
		<button class="action-button upload-button" type="button">📷 Image</button>
		<button class="action-button search-button" type="button">🔍 Recherche</button>
		<button class="action-button reason-button" type="button">🧠 Raisonne</button>
	</div>
</footer>
