<template>
  <div v-if="userProfile === 'admin'">
<q-card bordered>
  <q-card-section>
    <div class="text-h6 q-px-sm">Relatório Tickets</div>
  </q-card-section>
  <q-card-section class="q-pt-none">
    <fieldset class="rounded-all">
      <legend class="q-px-sm">Filtros</legend>
      <div class="row q-gutter-md items-center">
        <!-- Primeira linha com múltiplos filtros -->
        <div class="col-xs-12 col-md-3">
          <q-input
            outlined
            rounded
            dense
            v-model="searchQuery"
            label="Buscar por protocolo, nome, ticket ou celular"
          />
        </div>
        <div class="col-xs-6 col-md-3">
          <q-select
            outlined
            rounded
            dense
            v-model="statusFilter"
            :options="optionsStatus"
            option-value="value"
            option-label="label"
            emit-value
            map-options
            label="Status"
          />
        </div>
        <div class="col-xs-6 col-md-3">
          <DatePick
            dense
            rounded
            v-model="dateStartFilter"
            label="Data Inicial"
          />
        </div>
        <div class="col-xs-6 col-md-3">
          <DatePick
            dense
            rounded
            v-model="dateEndFilter"
            label="Data Final"
          />
        </div>
        <div class="col-xs-6 col-md-3">
          <q-select
            outlined
            rounded
            dense
            v-model="queueFilter"
            :options="queues"
            option-value="id"
            option-label="queue"
            emit-value
            map-options
            label="Fila"
          />
        </div>
        <div class="col-xs-6 col-md-3">
          <q-select
            outlined
            rounded
            dense
            v-model="attendantFilter"
            :options="attendants"
            option-value="id"
            option-label="name"
            emit-value
            map-options
            label="Atendente"
          />
        </div>
        <!-- Botões de ação -->
        <div class="col-xs-12 col-md-6 text-center">
          <q-btn
            class="q-mr-sm"
            color="primary"
            rounded
            label="Gerar"
            icon="refresh"
            @click="applyFilters"
          />
          <q-btn
            class="q-mr-sm"
            color="black"
            rounded
            label="Limpar Filtro"
            @click="clearFilters"
          />
          <q-btn
            color="warning"
            rounded
            label="Excel"
            @click="exportToXLSX"
          />
        </div>
      </div>
    </fieldset>
  </q-card-section>
</q-card>

    <div class="row">
      <div class="col-xs-12 q-mt-sm">
        <div
          class="tableLarge q-ma-sm q-markup-table q-table__container q-table__card q-table--cell-separator q-table--flat q-table--bordered q-table--no-wrap">
            <!-- Exibir estado de carregamento -->
    <div v-if="loading" class="loading">
      Carregando informações, por favor aguarde...
    </div>

        <!-- Tabela de atendimentos -->
        <table v-if="!loading" class="styled-table">
      <thead>
        <tr>
          <th>Ticket</th>
          <th>Status</th>
          <th>Nome</th>
          <th>Fila</th>
          <th>Atendente</th>
          <th>Horário Criação</th>
          <th>Horário Finalização</th>
          <th>Chat</th>
        </tr>
      </thead>
      <tbody>
        <tr v-for="ticket in filteredTickets" :key="ticket.id">
          <td>{{ ticket.id }}</td>
          <td><span :class="getStatusClass(ticket.status)">{{ ticket.status.toUpperCase() }}</span></td>
          <td>{{ ticket.name }}</td>
          <td>{{ ticket.queue }}</td>
          <td>{{ ticket.username || 'N/A' }}</td>
          <td>{{ formatDate(ticket.createdAt) }}</td>
          <td>{{ formatDate(ticket.updatedAt) }}</td>
          <td><a href="#" @click.prevent="visualizarChat(ticket.id)">VISUALIZAR CHAT</a></td>
        </tr>
      </tbody>
    </table>

    <div v-if="hasMore && !loading" class="pagination">
      <button @click="loadMoreTickets" class="filter-button">CARREGAR MAIS</button>
    </div>

        </div>
      </div>
    </div>

      <!-- Modal de Chat -->
      <ChatModal v-if="mostrarModal" :ticketId="String(ticketIdAtual)" @close="fecharChatModal" />
  </div>

</template>

<script>
import { ConsultarTickets } from 'src/service/tickets'
import { ListarFilas } from 'src/service/filas'
import { ListarUsuarios } from 'src/service/user'
import * as XLSX from 'xlsx'
import ChatModal from './ChatModal.vue'

export default {
  data () {
    return {
      optionsStatus: [
        { value: 'open', label: 'Aberto' },
        { value: 'pending', label: 'Pendente' },
        { value: 'closed', label: 'Fechado' }
      ],
      userProfile: 'user',
      contacts: [], // Lista de tickets
      searchQuery: '',
      statusFilter: '',
      dateStartFilter: '',
      dateEndFilter: '',
      queueFilter: '',
      queues: [], // Lista de filas
      attendantFilter: '',
      attendants: [], // Lista de atendentes
      pageNumber: 1,
      rowsPerPage: 30,
      hasMore: false,
      loading: false,
      mostrarModal: false,
      ticketIdAtual: null
    }
  },
  computed: {
    filteredTickets () {
      let tickets = this.contacts

      // Filtro por atendente localmente (comparando o userId com o attendantFilter)
      if (this.attendantFilter) {
        tickets = tickets.filter(ticket => {
          return ticket.userId === this.attendantFilter
        })
      }

      // Filtro por fila localmente (comparando o queueId com o queueFilter)
      if (this.queueFilter) {
        tickets = tickets.filter(ticket => {
          return ticket.queueId === this.queueFilter
        })
      }

      // Filtro por data localmente (usando o campo 'createdAt')
      if (this.dateStartFilter && this.dateEndFilter) {
        const startDate = new Date(this.dateStartFilter)
        const endDate = new Date(this.dateEndFilter)
        tickets = tickets.filter(ticket => {
          const ticketDate = new Date(ticket.createdAt)
          return ticketDate >= startDate && ticketDate <= endDate
        })
      }

      return tickets
    }
  },
  methods: {
    async consultarTickets () {
      this.loading = true
      try {
        const params = {
          searchParam: this.searchQuery || '',
          pageNumber: this.pageNumber,
          rowsPerPage: this.rowsPerPage,
          status: this.statusFilter ? [this.statusFilter] : ['pending', 'open', 'closed'],
          queuesIds: this.queueFilter ? [this.queueFilter] : [],
          date: this.dateStartFilter && this.dateEndFilter
            ? `${this.dateStartFilter}&${this.dateEndFilter}`
            : '',
          showAll: false,
          withUnreadMessages: false,
          isNotAssignedUser: false,
          includeNotQueueDefined: true
        }

        const { data } = await ConsultarTickets(params)

        if (data && data.tickets && data.tickets.length > 0) {
          this.contacts = this.pageNumber === 1 ? data.tickets : this.contacts.concat(data.tickets)
          this.hasMore = data.hasMore
        } else {
          this.hasMore = false
        }
      } catch (error) {
        console.error('Erro ao consultar tickets:', error)
      } finally {
        this.loading = false
      }
    },
    async loadQueuesAndAttendants () {
      try {
        const queuesData = await ListarFilas()
        this.queues = queuesData.data

        const attendantsData = await ListarUsuarios()
        this.attendants = attendantsData.data.users
      } catch (error) {
        console.error('Erro ao carregar filas e atendentes:', error)
      }
    },
    applyFilters () {
      this.pageNumber = 1
      this.consultarTickets()
    },
    loadMoreTickets () {
      this.pageNumber++
      this.consultarTickets().then(() => {
        // Espera a consulta de tickets terminar
        this.$nextTick(() => {
          // Rola até o final da página após carregar mais tickets
          window.scrollTo({
            top: document.documentElement.scrollHeight,
            behavior: 'auto' // ou 'smooth' para animação suave
          })
        })
      })
    },
    clearFilters () {
      this.searchQuery = ''
      this.statusFilter = ''
      this.dateStartFilter = ''
      this.dateEndFilter = ''
      this.queueFilter = ''
      this.attendantFilter = ''
      this.pageNumber = 1
      this.consultarTickets()
    },
    formatDate (dateString) {
      const date = new Date(dateString)
      return date.toLocaleString()
    },
    getStatusClass (status) {
      switch (status) {
        case 'open':
          return 'status-open'
        case 'closed':
          return 'status-closed'
        case 'pending':
          return 'status-pending'
        default:
          return ''
      }
    },
    visualizarChat (ticketId) {
      this.ticketIdAtual = ticketId
      this.mostrarModal = true
    },
    fecharChatModal () {
      this.mostrarModal = false
    },
    exportToXLSX () {
      const ws = XLSX.utils.json_to_sheet(this.filteredTickets)
      const wb = XLSX.utils.book_new()
      XLSX.utils.book_append_sheet(wb, ws, 'Tickets Filtrados')

      XLSX.writeFile(wb, 'tickets_filtrados.xlsx')
    }
  },
  mounted () {
    this.userProfile = localStorage.getItem('profile')
    this.consultarTickets()
    this.loadQueuesAndAttendants() // Carregar filas e atendentes ao montar o componente
  },
  components: {
    ChatModal
  }
}
</script>

<style scoped>
.container {
  padding: 20px;
  background-color: #202020;
  color: white;
  border-radius: 8px;
  box-shadow: 0px 0px 10px rgba(0, 0, 0, 0.5);
}

.filter-bar {
  display: flex;
  flex-wrap: wrap;
  justify-content: space-between;
  margin-bottom: 20px;
  gap: 10px;
}

.filter-input, .filter-select {
  padding: 10px;
  background-color: #333;
  border: 1px solid #555;
  color: white;
  border-radius: 5px;
  flex: 1;
}

.filter-button {
  padding: 10px 20px;
  background-color: #007bff;
  border: none;
  color: white;
  border-radius: 5px;
  cursor: pointer;
  transition: background-color 0.3s ease;
}

.filter-button:hover {
  background-color: #0056b3;
}

.export-button {
  background-color: #28a745;
}

.export-button:hover {
  background-color: #218838;
}

.styled-table {
  width: 100%;
  border-collapse: collapse;
  margin-top: 20px;
}

.styled-table th, .styled-table td {
  padding: 15px;
  text-align: left;
}

.styled-table th {
  background-color: #333;
  color: white;
}

.styled-table tr:nth-child(even) {
  background-color: #2b2b2b;
}

.styled-table td img.profile-pic {
  width: 40px;
  height: 40px;
  border-radius: 50%;
}

.status-open {
  background-color: #ffcc00;
  padding: 5px 10px;
  border-radius: 5px;
  color: black;
}

.status-closed {
  background-color: #4caf50;
  padding: 5px 10px;
  border-radius: 5px;
  color: white;
}

.status-pending {
  background-color: #ff9800;
  padding: 5px 10px;
  border-radius: 5px;
  color: white;
}

.loading {
  font-size: 18px;
  text-align: center;
  margin-top: 20px;
}

.styled-table a {
  color: #2196f3;
  text-decoration: none;
}

.styled-table a:hover {
  text-decoration: underline;
}

.pagination {
  text-align: center;
  margin-top: 20px;
}
</style>
