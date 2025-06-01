<script>
	import { onMount } from 'svelte';
	import PocketBase from 'pocketbase';
	import { Markdown } from 'svelte-exmarkdown';

	import Header from '$lib/components/header.svelte';
	import Footer from '$lib/components/footer.svelte';

	const pocketBase = new PocketBase(import.meta.env.VITE_POCKETBASE_URL);

	let conversations = [];
	let userMessage = '';
	let messages = [];
	let chatTitle = '';
	let activeConversationId = null;
	let drawerOpen = false;
	let bottomRef;

	$: messages, scrollToBottom();
	function scrollToBottom() {
		if (bottomRef) {
			bottomRef.scrollIntoView({ behavior: 'smooth' });
		}
	}

	function adjustHeight() {
		if (typeof window === 'undefined') return;
		const vh = window.innerHeight * 0.01;
		document.documentElement.style.setProperty('--vh', `${vh}px`);
	}

	onMount(() => {
		adjustHeight();
		window.addEventListener('resize', adjustHeight);
		window.addEventListener('load', adjustHeight);
		loadConversations();

		return () => {
			window.removeEventListener('resize', adjustHeight);
			window.removeEventListener('load', adjustHeight);
		};
	});

	function toggleDrawer() {
		drawerOpen = !drawerOpen;
	}

	async function loadConversations() {
		try {
			const list = await pocketBase.collection('conversation').getFullList();
			conversations = list.map((conv) => ({
				id: conv.id,
				title: conv.title,
				messages: []
			}));

			if (conversations.length > 0) {
				activeConversationId = conversations[0].id;
				await loadConversationMessages(activeConversationId);
			}
		} catch (error) {
			console.error('Erreur chargement conversations:', error);
		}
	}

	async function loadConversationMessages(conversationId) {
		try {
			const apiMessages = await pocketBase.collection('MessageUser_Ia').getFullList({
				filter: `(conversationId='${conversationId}')`,
				sort: 'created'
			});

			const conv = conversations.find((c) => c.id === conversationId);
			if (conv) {
				conv.messages = apiMessages.map((msg) => ({
					role: msg.is_ai_response ? 'assistant' : 'user',
					content: msg.content
				}));
				messages = [...conv.messages];
			}
		} catch (error) {
			console.error('Erreur chargement messages:', error);
		}
	}

	async function sendMessage() {
		if (!userMessage.trim()) return;

		const conv = conversations.find((c) => c.id === activeConversationId);
		if (!conv) return;

		const userMsg = { role: 'user', content: userMessage };
		userMessage = '';

		conv.messages.push(userMsg);
		messages = [...conv.messages];

		try {
			await pocketBase.collection('MessageUser_Ia').create({
				content: userMsg.content,
				is_ai_response: false,
				conversationId: activeConversationId
			});
		} catch (err) {
			console.error('Erreur enregistrement message utilisateur:', err);
		}

		try {
			const res = await fetch('https://api.mistral.ai/v1/chat/completions', {
				method: 'POST',
				headers: {
					Authorization: 'Bearer ' + import.meta.env.VITE_KeyApiMistral,
					'Content-Type': 'application/json'
				},
				body: JSON.stringify({
					model: 'mistral-medium',
					messages: conv.messages
				})
			});

			const data = await res.json();
			if (data?.choices?.length > 0) {
				const aiMsg = {
					role: 'assistant',
					content: data.choices[0].message.content
				};
				conv.messages.push(aiMsg);
				messages = [...conv.messages];

				await pocketBase.collection('MessageUser_Ia').create({
					content: aiMsg.content,
					is_ai_response: true,
					conversationId: activeConversationId
				});
			}
		} catch (error) {
			console.error('Erreur appel Mistral:', error);
		}
	}

	async function startNewConversation(title) {
		try {
			const record = await pocketBase.collection('conversation').create({ title });
			const newConv = { id: record.id, title: record.title, messages: [] };
			conversations = [...conversations, newConv];
			activeConversationId = newConv.id;
			messages = [];
		} catch (error) {
			console.error('Erreur création conversation:', error);
		}
	}

	async function selectConversation(id) {
		activeConversationId = id;
		await loadConversationMessages(id);
		drawerOpen = false;
	}

	async function deleteConversation(conversationId) {
		try {
			// 1. Récupère tous les messages liés à la conversation
			const toDelete = await pocketBase.collection('MessageUser_Ia').getFullList({
				filter: `(conversationId='${conversationId}')`,
				$autoCancel: false
			});

			// 2. Supprime tous les messages un par un (désactive l'auto-cancel)
			await Promise.all(
				toDelete.map((msg) =>
					pocketBase.collection('MessageUser_Ia').delete(msg.id, { $autoCancel: false })
				)
			);

			// 3. Supprime la conversation elle-même (désactive l'auto-cancel)
			await pocketBase.collection('conversation').delete(conversationId, { $autoCancel: false });

			// 4. Mets à jour la liste côté client
			conversations = conversations.filter((c) => c.id !== conversationId);

			// 5. Recharge une autre conversation si dispo
			if (conversations.length > 0) {
				activeConversationId = conversations[0].id;
				await loadConversationMessages(activeConversationId);
			} else {
				activeConversationId = null;
				messages = [];
			}
		} catch (error) {
			console.error('Erreur suppression conversation:', error);
		}
	}

	function handleNewChat(event) {
		event.preventDefault();
		if (!chatTitle.trim()) return;
		startNewConversation(chatTitle.trim());
		chatTitle = '';
		drawerOpen = false;
	}
</script>

<!-- Layout principal -->
<div class="app-layout">
	<!-- Menu latéral -->
	<aside class={`drawer ${drawerOpen ? 'open' : ''}`}>
		<h2 class="drawer-title">Conversations</h2>

		<!-- Liste des conversations -->
		<ul class="conversation-list" role="list">
			{#each conversations as conv}
				<li role="listitem" class="conversation-item" onclick={() => selectConversation(conv.id)}>
					💬 - {conv.title}
					<button
						aria-label="Supprimer une conversation"
						class="delete-icon"
						onclick={() => deleteConversation(conv.id)}
					>
						<svg
							xmlns="http://www.w3.org/2000/svg"
							height="24px"
							viewBox="0 -960 960 960"
							width="24px"
							fill="#fff"
							><path
								d="M280-120q-33 0-56.5-23.5T200-200v-520h-40v-80h200v-40h240v40h200v80h-40v520q0 33-23.5 56.5T680-120H280Zm400-600H280v520h400v-520ZM360-280h80v-360h-80v360Zm160 0h80v-360h-80v360ZM280-720v520-520Z"
							/></svg
						>
					</button>
				</li>
			{/each}
		</ul>

		<!-- Formulaire pour nouvelle conversation -->
		<form class="new-chat-form" onsubmit={handleNewChat}>
			<label for="chat-title" class="form-label">Nouvelle discussion</label>
			<input
				type="text"
				id="chat-title"
				name="chat-title"
				class="chat-title-input"
				bind:value={chatTitle}
				placeholder="Titre de la discussion..."
			/>
			<button type="submit" class="start-chat-button">Démarrer</button>
		</form>
	</aside>

	<!-- Contenu principal -->
	<main class="main-content">
		<!-- Header -->
		<Header {toggleDrawer} />

		<!-- Zone de messages -->
		<section class="chat-window">
			{#each messages as message}
				<div
					class="message-container"
					class:user={message.role === 'user'}
					class:assistant={message.role === 'assistant'}
				>
					{#if message.role === 'assistant'}
						<div class="message-bubble assistant-bubble">
							<div class="markdown-content">
								<Markdown md={message.content} />
							</div>
						</div>
					{:else}
						<div class="message-bubble user-bubble">
							{message.content}
						</div>
					{/if}
				</div>
			{/each}

			<!-- Élément invisible tout en bas qu'on va cibler pour scroller -->
			<div bind:this={bottomRef}></div>
		</section>

		<!-- Footer (input + boutons) -->
		<Footer bind:userMessage {sendMessage} />
	</main>
</div>

<!-- Backdrop pour fermer le menu sur mobile -->
{#if drawerOpen}
	<div
		class="backdrop"
		role="button"
		tabindex="0"
		onclick={toggleDrawer}
		onkeydown={(e) => (e.key === 'Enter' || e.key === ' ') && toggleDrawer()}
		aria-label="Fermer le menu"
		aria-hidden="true"
	></div>
{/if}
