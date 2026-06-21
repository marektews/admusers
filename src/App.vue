<template>
    <div class="container py-3">
        <header class="d-flex align-items-center justify-content-between mb-3">
            <h4 class="m-0">Konta użytkowników</h4>
            <button class="btn btn-primary" @click="openAdd" :disabled="accessError">
                <FontAwesomeIcon :icon="faPlus" class="me-2" />
                Dodaj użytkownika
            </button>
        </header>

        <div v-if="accessError" class="alert alert-danger">
            <FontAwesomeIcon :icon="faTriangleExclamation" class="me-2" />
            Brak dostępu — zaloguj się jako administrator z uprawnieniem „Użytkownicy".
        </div>

        <div v-if="errorMsg" class="alert alert-danger">
            <FontAwesomeIcon :icon="faTriangleExclamation" class="me-2" />
            {{ errorMsg }}
        </div>

        <table v-if="!accessError" class="table table-striped table-hover align-middle">
            <thead>
                <tr>
                    <th>#</th>
                    <th>Login</th>
                    <th>Imię</th>
                    <th>Nazwisko</th>
                    <th>Uprawnienia</th>
                    <th class="text-end">Akcje</th>
                </tr>
            </thead>
            <tbody>
                <tr v-for="(u, index) in users" :key="u.id">
                    <th>{{ index + 1 }}</th>
                    <td>{{ u.login }}</td>
                    <td>{{ u.fn }}</td>
                    <td>{{ u.ln }}</td>
                    <td>
                        <span
                            v-for="p in PERMS.filter(p => u.permissions?.[p.key])"
                            :key="p.key"
                            class="badge text-bg-secondary me-1"
                        >{{ p.label }}</span>
                    </td>
                    <td class="text-end text-nowrap">
                        <button class="btn btn-sm btn-outline-primary me-2" title="Edytuj" @click="openEdit(u)">
                            <FontAwesomeIcon :icon="faPen" />
                        </button>
                        <button class="btn btn-sm btn-outline-danger" title="Usuń" @click="askDelete(u)">
                            <FontAwesomeIcon :icon="faTrash" />
                        </button>
                    </td>
                </tr>
                <tr v-if="users.length === 0">
                    <td colspan="6" class="text-center text-muted">Brak kont.</td>
                </tr>
            </tbody>
        </table>

        <!-- Modal dodawania/edycji -->
        <div id="userModal" class="modal fade" tabindex="-1">
            <div class="modal-dialog">
                <div class="modal-content">
                    <div class="modal-header">
                        <h5 class="modal-title">{{ form.isEdit ? 'Edycja konta' : 'Nowe konto' }}</h5>
                        <button type="button" class="btn-close" data-bs-dismiss="modal" aria-label="Zamknij" />
                    </div>
                    <div class="modal-body">
                        <div class="mb-2">
                            <label class="form-label">Login</label>
                            <input class="form-control" v-model.trim="form.login" type="text" autocomplete="off">
                        </div>
                        <div class="row">
                            <div class="col mb-2">
                                <label class="form-label">Imię</label>
                                <input class="form-control" v-model.trim="form.fn" type="text">
                            </div>
                            <div class="col mb-2">
                                <label class="form-label">Nazwisko</label>
                                <input class="form-control" v-model.trim="form.ln" type="text">
                            </div>
                        </div>
                        <div class="mb-3">
                            <label class="form-label">
                                {{ form.isEdit ? 'Hasło (zostaw puste, by nie zmieniać)' : 'Hasło' }}
                            </label>
                            <input class="form-control" v-model="form.password" type="password" autocomplete="new-password">
                        </div>
                        <label class="form-label">Uprawnienia</label>
                        <div class="row">
                            <div v-for="p in PERMS" :key="p.key" class="col-6 form-check">
                                <input class="form-check-input" type="checkbox" :id="`perm_${p.key}`" v-model="form.permissions[p.key]">
                                <label class="form-check-label" :for="`perm_${p.key}`">{{ p.label }}</label>
                            </div>
                        </div>
                    </div>
                    <div class="modal-footer">
                        <button type="button" class="btn btn-secondary" data-bs-dismiss="modal">Anuluj</button>
                        <button type="button" class="btn btn-primary" :disabled="!canSave" @click="save">
                            <FontAwesomeIcon :icon="faFloppyDisk" class="me-2" />
                            Zapisz
                        </button>
                    </div>
                </div>
            </div>
        </div>

        <!-- Modal usuwania -->
        <div id="deleteModal" class="modal fade" tabindex="-1">
            <div class="modal-dialog">
                <div class="modal-content">
                    <div class="modal-header">
                        <h5 class="modal-title">Usuwanie konta</h5>
                        <button type="button" class="btn-close" data-bs-dismiss="modal" aria-label="Zamknij" />
                    </div>
                    <div class="modal-body">
                        Czy na pewno usunąć konto <b>{{ deleting?.login }}</b>?
                    </div>
                    <div class="modal-footer">
                        <button type="button" class="btn btn-secondary" data-bs-dismiss="modal">Anuluj</button>
                        <button type="button" class="btn btn-danger" @click="confirmDelete">
                            <FontAwesomeIcon :icon="faTrash" class="me-2" />
                            Usuń
                        </button>
                    </div>
                </div>
            </div>
        </div>
    </div>
</template>

<script setup>
import { ref, reactive, computed, onMounted } from 'vue'
import { Modal } from 'bootstrap'
import { FontAwesomeIcon } from '@fortawesome/vue-fontawesome'
import { faPlus, faPen, faTrash, faFloppyDisk, faTriangleExclamation } from '@fortawesome/free-solid-svg-icons'

const PERMS = [
    { key: 'is_sra', label: 'SRA' },
    { key: 'is_srp', label: 'SRP' },
    { key: 'is_pk', label: 'PK' },
    { key: 'is_rja', label: 'RJA' },
    { key: 'is_monitoring', label: 'Monitoring' },
    { key: 'is_users', label: 'Użytkownicy' },
    { key: 'is_limits', label: 'Limity' },
]

const users = ref([])
const accessError = ref(false)
const errorMsg = ref('')
const deleting = ref(null)

const emptyPermissions = () => Object.fromEntries(PERMS.map(p => [p.key, false]))

const form = reactive({
    isEdit: false,
    id: '',
    login: '',
    fn: '',
    ln: '',
    password: '',
    permissions: emptyPermissions(),
})

const canSave = computed(() => {
    if (form.login.length === 0) return false
    if (!form.isEdit && form.password.length === 0) return false
    return true
})

function flashError(msg) {
    errorMsg.value = msg
    setTimeout(() => { errorMsg.value = '' }, 5000)
}

function getModal(id) {
    return Modal.getOrCreateInstance(document.getElementById(id))
}

onMounted(loadUsers)

async function loadUsers() {
    try {
        const resp = await fetch('/api/users/all')
        if (resp.status === 401 || resp.status === 403) {
            accessError.value = true
            return
        }
        if (resp.status !== 200) {
            flashError('Nie udało się wczytać listy kont.')
            return
        }
        accessError.value = false
        const d = await resp.json()
        d.sort((a, b) => a.login.localeCompare(b.login))
        users.value = d
    } catch (err) {
        console.error('loadUsers:', err)
        flashError('Błąd połączenia z serwerem.')
    }
}

function openAdd() {
    form.isEdit = false
    form.id = ''
    form.login = ''
    form.fn = ''
    form.ln = ''
    form.password = ''
    form.permissions = emptyPermissions()
    getModal('userModal').show()
}

function openEdit(u) {
    form.isEdit = true
    form.id = u.id
    form.login = u.login
    form.fn = u.fn
    form.ln = u.ln
    form.password = ''
    form.permissions = { ...emptyPermissions(), ...(u.permissions || {}) }
    getModal('userModal').show()
}

async function save() {
    const url = form.isEdit ? '/api/users/update' : '/api/users/create'
    const body = {
        login: form.login,
        password: form.password,
        fn: form.fn,
        ln: form.ln,
        permissions: form.permissions,
    }
    if (form.isEdit) body.id = form.id

    try {
        const resp = await fetch(url, {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify(body),
        })
        if (resp.status === 200 || resp.status === 201) {
            getModal('userModal').hide()
            await loadUsers()
        } else if (resp.status === 409) {
            flashError('Konto o takim loginie już istnieje.')
        } else if (resp.status === 403) {
            flashError('Brak uprawnień do zarządzania kontami.')
        } else {
            flashError('Zapis konta nie powiódł się.')
        }
    } catch (err) {
        console.error('save:', err)
        flashError('Błąd połączenia z serwerem.')
    }
}

function askDelete(u) {
    deleting.value = u
    getModal('deleteModal').show()
}

async function confirmDelete() {
    if (!deleting.value) return
    try {
        const resp = await fetch(`/api/users/delete/${deleting.value.id}`)
        getModal('deleteModal').hide()
        if (resp.status === 200) {
            await loadUsers()
        } else if (resp.status === 400) {
            flashError('Nie można usunąć własnego konta.')
        } else {
            flashError('Usuwanie konta nie powiodło się.')
        }
    } catch (err) {
        console.error('confirmDelete:', err)
        flashError('Błąd połączenia z serwerem.')
    } finally {
        deleting.value = null
    }
}
</script>

<style>
#app {
    -webkit-font-smoothing: antialiased;
    -moz-osx-font-smoothing: grayscale;
}
</style>
