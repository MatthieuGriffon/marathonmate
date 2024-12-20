<script lang="ts">
    import JoinMarathonModal from '$lib/components/JoinMarathonModal.svelte';
    import AddFilmModal from '$lib/components/AddFilmModal.svelte';
    import { onMount } from 'svelte';
    import type { PageData } from './$types';
    let marathons: Marathon[] = [];
    let showModal = false;
    let marathonToDelete: string | null = null;
    let userToDelete: { marathonId: string; userId: string } | null = null;
    let showJoinModal = false;
    let showAddFilmModal = false;
    let selectedMarathonId: string | null = null;
    let showNotification = false; //

    export let data: PageData;
    let { user} = data;
    onMount(loadMarathons);

    // Fonction pour charger les marathons depuis le backend
    async function loadMarathons() {
    try {
        const response = await fetch('/marathons');
        if (response.ok) {
            const marathonsData = await response.json();

            // Charger les films et les participants pour chaque marathon dès le chargement
            marathons = await Promise.all(
                marathonsData.map(async (marathon: Marathon) => {
                    const invitedUsers = await loadInvitedUsers(marathon.id);
                    return { ...marathon, invitedUsers };
                })
            );
        } else {
            console.error('Erreur lors du chargement des marathons');
        }
    } catch (error) {
        console.error('Erreur réseau:', error);
    }
}
    function openAddFilmModal(marathonId: string) {
        selectedMarathonId = marathonId;
        showAddFilmModal = true;
    }
    async function handleAddFilms(event: CustomEvent<{ films: any[] }>) {
        const { films } = event.detail;
        const response = await fetch(`/marathons/${selectedMarathonId}/films/add`, {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify({ films })
        });
        if (response.ok) {
            loadMarathons(); // recharge la liste avec les nouveaux films
        }
        showAddFilmModal = false;
        selectedMarathonId = null;
    }
    async function toggleMarathon(marathonId: string) {
        marathons = await Promise.all(
            marathons.map(async (marathon) => {
                if (marathon.id === marathonId) {
                    const isOpen = !marathon.isOpen;
                    const films = isOpen ? await loadMarathonMovies(marathonId) : marathon.films || [];
                    const invitedUsers = isOpen ? await loadInvitedUsers(marathonId) : marathon.invitedUsers || [];
                    return { ...marathon, isOpen, films, invitedUsers };
                }
                return marathon;
            })
        );
    }
    async function loadMarathonMovies(marathonId: string) {
        const response = await fetch(`/marathons/${marathonId}/films`);
        return response.ok ? await response.json() : [];
    }
    async function loadInvitedUsers(marathonId: string) {
        const response = await fetch(`/marathons/${marathonId}/invited-users`);
        return response.ok ? await response.json() : [];
    }
    async function handleJoinSuccess(event: CustomEvent<{ marathonId: string }>) {
    const { marathonId } = event.detail;
    await loadMarathons(); // Recharge la liste des marathons
    showJoinModal = false; // Ferme la modal une fois le marathon rejoint
}
function copyInvitationCode(code: string) {
        navigator.clipboard.writeText(code)
            .then(() => {
                showNotification = true;
                setTimeout(() => {
                    showNotification = false;
                }, 3000); // Notification disparaît après 3 secondes
            })
            .catch((error) => console.error('Erreur lors de la copie du code :', error));
    }
    function openJoinModal() { showJoinModal = true; }
    function openDeleteModal(marathonId: string) { marathonToDelete = marathonId; showModal = true; }
    function cancelDelete() { marathonToDelete = null; showModal = false; }
    async function confirmDeleteMarathon() {
        if (marathonToDelete) {
            const response = await fetch(`/marathons/${marathonToDelete}/delete`, { method: 'DELETE' });
            if (response.ok) {
                marathons = marathons.filter(marathon => marathon.id !== marathonToDelete);
                marathonToDelete = null;
                showModal = false;
            } else {
                console.error("Erreur lors de la suppression du marathon");
            }
        }
    }
    function openUserDeleteModal(marathonId: string, userId: string) {
        userToDelete = { marathonId, userId };
        showModal = true;
    }
    async function confirmDeleteUser() {
        if (userToDelete) {
            const { marathonId, userId } = userToDelete;
            const response = await fetch(`/marathons/${marathonId}/invited-users/${userId}/delete`, { method: 'DELETE' });
            if (response.ok) {
                marathons = marathons.map(marathon => {
                    if (marathon.id === marathonId) {
                        return {
                            ...marathon,
                            invitedUsers: marathon.invitedUsers.filter(user => user.id !== userId)
                        };
                    }
                    return marathon;
                });
                userToDelete = null;
                showModal = false;
            } else {
                console.error("Erreur lors de la suppression de l'invité");
            }
        }
    }
    async function removeMovieFromMarathon(marathonId: string, movieId: string) {
    const response = await fetch(`/marathons/${marathonId}/films/${movieId}/delete`, { method: 'DELETE' });

    if (response.ok) {
        // Met à jour la liste des films dans le frontend après la suppression
        marathons = marathons.map(marathon => {
            if (marathon.id === marathonId) {
                return {
                    ...marathon,
                    films: marathon.films.filter(film => film.id !== movieId)
                };
            }
            return marathon;
        });
    } else {
        console.error("Erreur lors de la suppression du film");
    }
}
</script>

<div class="marathons-list">
    <h1>Liste de vos Marathons</h1> 
    <button class="join-button" on:click={openJoinModal}>Rejoindre un Marathon</button>
    {#if marathons.length === 0}
        <p>Vous n'avez pas encore de marathons.</p>
    {/if}

    {#each marathons as marathon}
        <div class="marathon-card">
            <div class="marathon-header">
                <h3>{marathon.name} <span class="organizer"> - Organisé par {marathon.organizerName}</span></h3>
                <div class="invited-users">
                    <h4>Participants :</h4>
                    <ul>
                        {#each marathon.invitedUsers as user}
                            <li>
                                {user.name} - {user.status}
                                <button on:click={() => openUserDeleteModal(marathon.id, user.id)} class="remove-user-button">
                                    Supprimer
                                </button>
                            </li>
                        {/each}
                    </ul>
                </div>
                <button on:click={() => openAddFilmModal(marathon.id)}>Ajouter des Films</button>               
                <button on:click={() => toggleMarathon(marathon.id)}>
                    {marathon.isOpen ? 'Masquer les films' : 'Voir les films'}
                </button>
            </div>
             <!-- Affichage du code d'invitation -->
             <div class="invitation-code">
                <p><strong>Code d'invitation :</strong> {marathon.invitationCode}</p>
                <button on:click={() => copyInvitationCode(marathon.invitationCode)}>Copier le code</button>
            </div>
            {#if marathon.isOpen}
                <div class="films-list">
                    <h4>Films :</h4>
                    <ul>
                        {#each marathon.films as film}
                        <li>
                            <img src={film.posterUrl} alt="Affiche de {film.title}" class="film-poster" />
                            <div class="film-info">
                                <p><strong>{film.title}</strong></p>
                                <p>Date de sortie : {new Date(film.releaseDate).toLocaleDateString('fr-CA')}</p>
                                <button on:click={() => removeMovieFromMarathon(marathon.id, film.id)} class="remove-movie-button">
                                    Supprimer
                                </button>
                            </div>
                        </li>
                        {/each}
                    </ul>
                </div>
               
            {/if}
             <!-- Bouton visible uniquement pour l'organisateur -->
             {#if marathon.organizerId === user?.id}
             <button class="delete" on:click={() => openDeleteModal(marathon.id)}>Supprimer le marathon</button>
         {/if}
        </div>
    {/each}
</div>

<JoinMarathonModal
    isOpen={showJoinModal}
    on:close={() => showJoinModal = false}
    on:joinSuccess={handleJoinSuccess}
/>

{#if showModal}
    <div class="modal-overlay">
        <div class="modal">
            <h2>Confirmer la suppression</h2>
            <p>Êtes-vous sûr de vouloir supprimer ce marathon ? Cette action est irréversible.</p>
            <div class="modal-buttons">
                <button class="confirm" on:click={confirmDeleteMarathon}>Oui, supprimer</button>
                <button class="cancel" on:click={cancelDelete}>Annuler</button>
            </div>
        </div>
    </div>
{/if}

{#if showModal && userToDelete}
    <div class="modal-overlay">
        <div class="modal">
            <h2>Confirmer la suppression</h2>
            <p>Êtes-vous sûr de vouloir supprimer cet utilisateur invité ?</p>
            <div class="modal-buttons">
                <button class="confirm" on:click={confirmDeleteUser}>Oui, supprimer</button>
                <button class="cancel" on:click={cancelDelete}>Annuler</button>
            </div>
        </div>
    </div>
{/if}
{#if showAddFilmModal}
    <AddFilmModal on:close={() => showAddFilmModal = false} on:addfilms={handleAddFilms} />
{/if}
{#if showNotification}
    <div class="notification">
        Code d'invitation copié !
    </div>
{/if}
<style>

.invitation-code {
        display: flex;
        align-items: center;
        gap: 1rem;
        margin-top: 1rem;
    }
    .invitation-code button {
        padding: 0.3rem 0.6rem;
        font-size: 0.9rem;
        border: none;
        background-color: #4285f4;
        color: white;
        border-radius: 4px;
        cursor: pointer;
    }
    .notification {
        position: fixed;
        top: 50%;
        left: 50%;
        transform: translate(-50%, -50%);
        background-color: #4caf50;
        color: white;
        padding: 1rem 2rem;
        border-radius: 8px;
        box-shadow: 0px 4px 6px rgba(0, 0, 0, 0.1);
        animation: fade-in-out 3s;
        z-index: 1000;
    }
    @keyframes fade-in-out {
        0% { opacity: 0; transform: translate(-50%, -60%); }
        10% { opacity: 1; transform: translate(-50%, -50%); }
        90% { opacity: 1; transform: translate(-50%, -50%); }
        100% { opacity: 0; transform: translate(-50%, -60%); }
    }
.marathons-list {
        max-width: 600px;
        margin: 2rem auto;
        padding: 1rem;
    }

    .marathon-card {
        border: 1px solid #ddd;
        border-radius: 8px;
        padding: 1rem;
        margin-bottom: 1rem;
        background-color: #f9f9f9;
    }

    .marathon-card h3 {
        margin: 0;
        font-size: 1.25rem;
    }

    .marathon-card p {
        margin: 0.5rem 0;
    }

    .films-list {
        margin-top: 1rem;
    }

    .films-list h4 {
        margin-bottom: 0.5rem;
    }

    .films-list ul {
        list-style: none;
        padding: 0;
        display: flex;
        flex-wrap: wrap;
        gap: 1rem;
    }

    .films-list li {
        display: flex;
        align-items: center;
        gap: 0.5rem;
    }

    .film-poster {
        width: 50px;
        height: 75px;
        object-fit: cover;
        border-radius: 4px;
    }

    .film-info {
        display: flex;
        flex-direction: column;
    }

    button {
        padding: 0.5rem 1rem;
        font-size: 1rem;
        border: none;
        border-radius: 4px;
        cursor: pointer;
        transition: background-color 0.3s;
    }

    button.delete {
        background-color: #e74c3c;
        color: white;
    }

    button:not(.delete) {
        background-color: #737973;
        color: white;
    }

    .modal-overlay {
        position: fixed;
        top: 0;
        left: 0;
        right: 0;
        bottom: 0;
        background: rgba(0, 0, 0, 0.5);
        display: flex;
        justify-content: center;
        align-items: center;
        z-index: 1000;
    }

    .modal {
        background: white;
        padding: 2rem;
        border-radius: 8px;
        max-width: 400px;
        width: 90%;
        text-align: center;
    }
    .remove-user-button {
        background-color: #e74c3c;
        color: white;
        border: none;
        border-radius: 4px;
        padding: 0.2rem 0.5rem;
        cursor: pointer;
        margin: 0.5rem;
    }
    .remove-user-button:hover {
        background-color: #e74c3c;
    }
    .join-button {
        margin: 0.5rem;
        padding: 0.8rem ;
        font-size: 1rem;
        background-color: #4285f4;
        color: white;
        border: none;
        border-radius: 4px;
        cursor: pointer;
        transition: background-color 0.3s;
    }

    .join-button:hover {
        background-color: #27282b; /* Couleur survolée */
    }

    .join-button:active {
        background-color: #285a9b; /* Couleur lorsque le bouton est actif */
    }
</style>