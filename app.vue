<template>
  <main class="min-h-screen flex items-center justify-center bg-gray-50">
    <form action="#" class="w-full max-w-lg p-6 bg-white rounded-xl shadow space-y-6">
      <div>
        <label for="gender" class="block text-sm font-medium text-gray-700">Genre</label>
        <select name="gender" id="gender" v-model="selectedGender" class="mt-1 w-full rounded border-gray-300">
          <option value="men">Homme</option>
          <option value="women">Femme</option>
        </select>
      </div>

      <div>
        <label for="division" class="block text-sm font-medium text-gray-700">Division</label>
        <select name="division" id="division" v-model="selectedDivision" class="mt-1 w-full rounded border-gray-300">
          <option v-for="division in divisions" :key="division" :value="division">
            {{ division }}
          </option>
        </select>
      </div>

      <div>
        <label for="series" class="block text-sm font-medium text-gray-700">Série</label>
        <select name="series" id="series" v-model="selectedSeries" class="mt-1 w-full rounded border-gray-300">
          <option value="a">Série A</option>
          <option value="b">Série B</option>
        </select>
      </div>

      <div>
        <label for="team_number" class="block text-sm font-medium text-gray-700">
          Nombre d'équipes dans le championnat
        </label>
        <input type="number" id="team_number" v-model="teamCount" min="1" class="mt-1 w-full rounded border-gray-300"/>
      </div>

      <div v-for="i in teamOptions" :key="i" class="space-y-1">
        <label :for="'team_' + i" class="block text-sm font-medium text-gray-700">
          Équipe {{ i + 1 }}
        </label>
        <select
            :name="'team_' + i"
            :id="'team_' + i"
            v-model="selectedTeams[i]"
            class="w-full rounded border-gray-300"
        >
          <option disabled value="">-- Choisir un club --</option>
          <option v-for="club in clubsList" :key="club.name" :value="club">
            {{ club.name }}
          </option>
        </select>
        <div v-if="selectedTeams[i]" class="text-sm text-gray-500 pl-2">
          Préfère jouer le <strong>{{ selectedTeams[i].preferredDay }}</strong> à
          <strong>{{ selectedTeams[i].preferredHour }}</strong>
        </div>
      </div>

      <div v-if="weekendCount > 0" class="space-y-4">
        <h2 class="text-md font-semibold text-gray-700">Choix des week-ends de rencontre</h2>
        <div v-for="(wk, index) in weekendWeeks" :key="index">
          <label :for="'week_' + index" class="block text-sm font-medium text-gray-700">
            Week-end {{ index + 1 }} – Semaine de l'année
          </label>
          <input
              type="number"
              min="1"
              max="52"
              class="mt-1 w-full rounded border-gray-300"
              v-model="weekendWeeks[index]"
              :id="'week_' + index"
          />
        </div>
      </div>

      <button
          type="button"
          class="w-full bg-blue-600 text-white py-2 px-4 rounded hover:bg-blue-700"
          @click="submit"
      >
        Valider
      </button>
    </form>
  </main>
</template>

<script setup>
import {ref, computed, watch} from 'vue'
import {club} from '~/public/data/clubs.js'
import * as XLSX from 'xlsx'

const selectedGender = ref('men')
const selectedSeries = ref('a')
const selectedDivision = ref('p1')
const teamCount = ref(0)
const selectedTeams = ref([])
const weekendWeeks = ref([])

const weekendCount = computed(() => {
  const map = {
    4: 6,
    6: 10,
    8: 14,
    10: 18,
    12: 22,
    14: 26
  }
  return map[teamCount.value] || 0
})

watch(weekendCount, (count) => {
  weekendWeeks.value = Array.from({length: count}, (_, i) => weekendWeeks.value[i] ?? '')
})

const divisions = computed(() => {
  return Object.keys(club[selectedGender.value])
})

const clubsList = computed(() => {
  return club[selectedGender.value][selectedDivision.value]?.[selectedSeries.value] || []
})


const teamOptions = computed(() => {
  return Array.from({length: teamCount.value}, (_, index) => index)
})

watch([selectedGender, selectedDivision, selectedSeries], () => {
  teamCount.value = 0
  selectedTeams.value = []
})

watch(teamCount, (newCount) => {
  selectedTeams.value = Array.from({length: newCount}, (_, i) => selectedTeams.value[i] || '')
})

function getWeekendDatesFromWeekNumber(week, year = new Date().getFullYear()) {
  const simple = new Date(year, 0, 1 + (week - 1) * 7)
  const dayOfWeek = simple.getDay()
  const monday = new Date(simple)
  if (dayOfWeek <= 4) {
    monday.setDate(simple.getDate() - dayOfWeek + 1)
  } else {
    monday.setDate(simple.getDate() + 8 - dayOfWeek)
  }

  const friday = new Date(monday)
  friday.setDate(monday.getDate() + 4)
  const saturday = new Date(monday)
  saturday.setDate(monday.getDate() + 5)
  const sunday = new Date(monday)
  sunday.setDate(monday.getDate() + 6)

  const format = (d) =>
      `${String(d.getDate()).padStart(2, '0')}/${String(d.getMonth() + 1).padStart(2, '0')}/${String(d.getFullYear()).slice(-2)}`

  return {
    friday: format(friday),
    saturday: format(saturday),
    sunday: format(sunday)
  }
}

const submit = () => {
  const referenceYear = new Date().getFullYear()
  const baseWeek = weekendWeeks.value[0]
  let currentYear = referenceYear
  const result = {
    gender: selectedGender.value,
    division: selectedDivision.value,
    teamCount: teamCount.value,
    teams: selectedTeams.value.map((team, i) => ({
      index: i,
      name: team?.name ?? 'Non défini',
      preferredDay: team?.preferredDay ?? 'N/A',
      preferredHour: team?.preferredHour ?? 'N/A'
    })),

    weekendsPreferences: weekendWeeks.value.map((week, index) => {
      if (week < baseWeek && index !== 0) {
        currentYear = referenceYear + 1
      }
      const dates = getWeekendDatesFromWeekNumber(week, currentYear)

      return {
        week,
        dates,
        teamsPreferences: selectedTeams.value.map((team, i) => {
          let matchDate = ''
          switch (team?.preferredDay) {
            case 'vendredi':
              matchDate = dates.friday
              break
            case 'samedi':
              matchDate = dates.saturday
              break
            case 'dimanche':
              matchDate = dates.sunday
              break
            default:
              matchDate = 'N/A'
          }
          return {
            index: i,
            name: team?.name ?? 'Non défini',
            preferredDay: team?.preferredDay ?? 'N/A',
            preferredHour: team?.preferredHour ?? 'N/A',
            matchDate
          }
        })
      }
    })
  }

  console.log(result)

  fetch('/data/time_grid.xls')
      .then((res) => res.arrayBuffer())
      .then((buffer) => {
        const workbook = XLSX.read(buffer, {type: 'array'})
        const sheetName = `Grille ${result.teamCount}_A`
        const sheet = workbook.Sheets[sheetName]

        if (sheet) {
          const data = XLSX.utils.sheet_to_json(sheet, {header: 1})
          console.log(`Données de ${sheetName}:`, data)
          const wb = XLSX.utils.book_new()
          const rows = []
          rows.push(['Genre', 'Division', 'Serie', 'Club domicile', 'Club extérieur', 'Date', 'Heure'])

          for (let i = 0; i < data.length; i++) {
            for (let j = 0; j < data[i].length; j++) {
              if (j % 3 === 0) {
                const regex = /Eq\.(\d+)/;
                const homeTeamNumber = data[i][j + 1].match(regex)[1]
                const extTeamNumber = data[i][j + 2].match(regex)[1]
                const homeTeam = result.teams[homeTeamNumber - 1]
                const extTeam = result.teams[extTeamNumber - 1]
                const weekmatch = data[i][j]
                const weekendPref = result.weekendsPreferences[weekmatch - 1]
                console.log()
                const dates = weekendPref?.dates ?? {}

                let matchDate = ''
                switch (homeTeam?.preferredDay) {
                  case 'vendredi':
                    matchDate = dates.friday
                    break
                  case 'samedi':
                    matchDate = dates.saturday
                    break
                  case 'dimanche':
                    matchDate = dates.sunday
                    break
                  default:
                    matchDate = 'N/A'
                }
                rows.push([selectedGender.value, selectedDivision.value, selectedSeries.value, homeTeam.name, extTeam.name, matchDate, homeTeam.preferredHour])
              }
            }
          }


          // Transformer en feuille et l’ajouter au classeur
          const ws = XLSX.utils.aoa_to_sheet(rows)
          XLSX.utils.book_append_sheet(wb, ws, 'Rencontres')

          // Sauvegarder dans un fichier .xlsx
          XLSX.writeFile(wb, 'grille_rencontres.xlsx')

        } else {
          console.error('Feuille non trouvée !')
        }
      })
}
</script>

