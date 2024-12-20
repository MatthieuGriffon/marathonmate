<script lang="ts">
    import { goto, invalidateAll } from '$app/navigation';
    import { page } from '$app/stores';
    import { enhance } from '$app/forms';
    $: console.log("URL de la photo de profil 01 : ", user.profile_picture);
    $: user = $page.data.user;
    let showPasswordChange = false;
    let oldPassword = '';
    let newPassword = '';
    let confirmPassword = '';
    let passwordChangeMessage = '';
    let showDeleteConfirmation = false;
    let deleteMessage = '';
    let profilePicture: File | null = null;
    let uploadMessage = '';

    async function handleLogout() {
        await fetch('/logout', { method: 'POST' });
        await invalidateAll();
        goto('/');
    }

    async function handleChangePassword() {
        if (newPassword !== confirmPassword) {
            passwordChangeMessage = 'Les nouveaux mots de passe ne correspondent pas.';
            return;
        }

        const response = await fetch('/profile/change-password', {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify({ oldPassword, newPassword })
        });

        if (response.ok) {
            passwordChangeMessage = 'Mot de passe mis à jour avec succès.';
        } else {
            const data = await response.json();
            passwordChangeMessage = data.error || 'Erreur lors de la mise à jour du mot de passe.';
        }
    }

    async function handleDeleteAccount() {
    try {
        const response = await fetch('/profile/delete-account', {
            method: 'POST'
        });

        if (response.status === 204) {
            deleteMessage = 'Compte supprimé avec succès.';
            await invalidateAll();
            goto('/');
        } else {
            const data = await response.json();
            deleteMessage = data.error || 'Erreur lors de la suppression du compte.';
        }
    } catch (error) {
        console.error('Erreur lors de la suppression du compte :', error);
        deleteMessage = 'Une erreur est survenue. Veuillez réessayer.';
    }
}
    async function handleFileUpload() {
    if (!profilePicture) {
        uploadMessage = 'Please select a file.';
        return;
    }

    const formData = new FormData();
    formData.append('profilePicture', profilePicture);

    try {
        const response = await fetch('/upload-profile-picture', {
            method: 'POST',
            body: formData,
        });

        if (response.ok) {
            const data = await response.json();
            // Mets à jour directement la photo de profil sans invalider
            user.profile_picture = `${data.profilePictureUrl}?t=${new Date().getTime()}`;
            console.log('Profile picture uploaded:', user.profile_picture);
            uploadMessage = 'Upload successful!';
            
            // Optionnel : Utiliser invalidateAll si les données ne sont pas mises à jour correctement
            // await invalidateAll(); 
        } else {
            uploadMessage = 'Upload failed. Please try again.';
        }
    } catch (error) {
        console.error(error);
        uploadMessage = 'An error occurred during upload.';
    }
}

</script>

{#if user}
<section class="profile">
    <h1>Profil de {user.name}</h1>
    <div class="profile-info">
        {#if user.profile_picture}
            <img src="{user.profile_picture}" alt="Photo de profil de {user.name}" class="profile-picture" />
        {/if}
        <p><strong>Nom :</strong> {user.name}</p>
        <p><strong>Email :</strong> {user.email}</p>
    </div>
    
    {#if !user.oauthProvider}
        <!-- svelte-ignore a11y_click_events_have_key_events -->
        <!-- svelte-ignore a11y_no_noninteractive_element_interactions -->
         
        <div class="photo-upload-card">
            <form on:submit|preventDefault={handleFileUpload}>
                <label for="profilePicture">Changer votre photo de profil :</label>
                <input type="file" id="profilePicture" accept="image/jpeg, image/png" on:change={(e) => { const target = e.target as HTMLInputElement; if (target.files) profilePicture = target.files[0]; }} />
                <button type="submit">Upload</button>
            </form>
            <p>{uploadMessage}</p>
        </div>
        <!-- svelte-ignore a11y_click_events_have_key_events -->
        <!-- svelte-ignore a11y_no_noninteractive_element_interactions -->
        <p class="change-password-link" on:click={() => showPasswordChange = !showPasswordChange}>
            {showPasswordChange ? 'Annuler' : 'Changer de mot de passe'}
        </p>
        
        {#if showPasswordChange}
            <div class="password-change-card">
                <label>
                    Ancien mot de passe :
                    <input type="password" bind:value={oldPassword} required />
                </label>
                <label>
                    Nouveau mot de passe :
                    <input type="password" bind:value={newPassword} required />
                </label>
                <label>
                    Confirmez le nouveau mot de passe :
                    <input type="password" bind:value={confirmPassword} required />
                </label>
                <button on:click={handleChangePassword}>Mettre à jour le mot de passe</button>
                {#if passwordChangeMessage}
                    <p class="message">{passwordChangeMessage}</p>
                {/if}
            </div>
        {/if}
    {/if}
    
    <button on:click={handleLogout}>Se déconnecter</button>
    <button on:click={() => showDeleteConfirmation = !showDeleteConfirmation} class="delete-account-button">
        Supprimer le compte
    </button>
    
    {#if showDeleteConfirmation}
        <div class="delete-confirmation">
            <p>Êtes-vous sûr de vouloir supprimer votre compte ? Cette action est irréversible.</p>
            <button on:click={handleDeleteAccount} class="confirm-delete">Oui, supprimer</button>
            <button on:click={() => showDeleteConfirmation = false} class="cancel-delete">Annuler</button>
        </div>
    {/if}
    {#if deleteMessage}
        <p class="delete-message">{deleteMessage}</p>
    {/if}
</section>
{:else}
    <p>Veuillez vous connecter pour accéder à votre profil.</p>
{/if}



<style>
    .profile {
    max-width: 400px;
    margin: 1rem auto;
    padding: 2rem;
    text-align: center;
    background-color: #f5f5f586;
    border-radius: 8px;
    box-shadow: 0px 4px 8px rgba(0, 0, 0, 0.2);
    display: flex;
    flex-direction: column;
    align-items: center;
}
.profile-picture {
    border-radius: 50%;
    width: 100px;
    height: 100px;
    object-fit: cover;
    margin-bottom: 1rem;
}
.profile-info {
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 0.5rem;
}
.password-change-card {
    max-width: 100%;
    background-color: #ffffff;
    border: 1px solid #ddd;
    border-radius: 8px;
    padding: 1.5rem;
    margin-top: 1.5rem;
    box-shadow: 0px 2px 4px rgba(0, 0, 0, 0.1);
    font-size: 0.9rem;
}
.password-change-card label {
    font-weight: bold;
    margin-top: 0.5rem;
    font-size: 0.9rem;
}
.password-change-card input[type="password"] {
    width: 100%;
    padding: 0.4rem;
    margin: 0.5rem 0;
    border-radius: 4px;
    font-size: 0.9rem;
}
.password-change-card button {
    margin-top: 1rem;
    padding: 0.4rem 1rem;
    font-size: 0.9rem;
    background-color: #333;
}
    .profile-info {
        display: flex;
        flex-direction: column;
        align-items: center;
        gap: 0.5rem;
    }
    .message {
    margin-top: 0.5rem;
    color: green;
}
button {
    margin-top: 1.5rem;
    padding: 0.5rem 1rem;
    font-size: 1rem;
    background-color: #333;
    color: white;
    border: none;
    border-radius: 4px;
    cursor: pointer;
    transition: background-color 0.3s ease;
}
button:hover {
    background-color: #555;
}
.change-password-link {
    margin-top: 1rem;
    font-size: 0.9rem;
    color: #ffffff;
    cursor: pointer;
    background-color: #333;
    padding:0.5rem;
    border-radius: 4px;
}
.change-password-link:hover {
    background-color: #555;
}
.photo-upload-card {
    max-width: 100%;
    background-color: #70696993;
    border: 1px solid #ddd;
    border-radius: 8px;
    padding: 0.5rem;
    margin-top: 1rem;
    box-shadow: 0px 2px 4px rgba(0, 0, 0, 0.1);
    font-size: 0.9rem;
    text-align: center;
}
.photo-upload-card label {
    font-weight: bold;
    margin-bottom: 1rem;
    display: block;
}
.photo-upload-card input[type="file"] {
    margin: 1rem 0;
}
.photo-upload-card button {
    margin-top: 1rem;
    padding: 0.4rem 1rem;
    font-size: 0.9rem;
    background-color: #333;
    color: rgba(255, 255, 255, 0.993);
    border: none;
    border-radius: 4px;
    cursor: pointer;
    transition: background-color 0.3s ease;
}
.photo-upload-card button:hover {
    background-color: #555;
}
.photo-upload-card p {
    margin-top: 0.5rem;
    font-size: 0.9rem;
    color: green;
}
</style>
