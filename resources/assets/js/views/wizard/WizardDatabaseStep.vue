<template>
  <sw-wizard-step
    :title="$t('wizard.database.database')"
    :description="$t('wizard.database.desc')"
  >
    <base-loader v-if="isFetching" :show-bg-overlay="true" />
    <component
      :is="database_connection"
      :config-data="databaseData"
      :is-loading="isLoading"
      :is-fetching="isFetching"
      @on-change-driver="getDatabaseConfig"
      @submit-data="next"
    />
  </sw-wizard-step>
</template>

<script>
import { validationMixin } from 'vuelidate'
import Mysql from './database/MysqlDatabase'
import Pgsql from './database/PgsqlDatabase'
import Sqlite from './database/SqliteDatabase'
import Sqlsrv from './database/SqlsrvDatabase'
import { mapActions } from 'vuex'
export default {
  components: {
    Mysql,
    Pgsql,
    Sqlite,
    Sqlsrv,
  },
  data() {
    return {
      databaseData: {
        database_connection: 'mysql',
      },
      isLoading: false,
      isFetching: false,
      database_connection: 'mysql',
      connections: ['sqlite', 'mysql', 'pgsql', 'sqlsrv'],
    }
  },
  created() {
    this.getDatabaseConfig(this.database_connection)
  },
  methods: {
    ...mapActions('notification', ['showNotification']),
    async getDatabaseConfig(connection) {
      this.isLoading = this.isFetching = true

      let params = {
        connection,
      }

      let response = await window.axios.get(
        '/api/v1/onboarding/database/config',
        { params }
      )

      if (response.data.success) {
        this.databaseData = response.data.config
        this.database_connection = connection
        this.databaseData.database_connection = connection
        this.isLoading = this.isFetching = false
      }
    },
    async next(databaseData) {
      this.isLoading = this.isFetching = true
      try {
        await window.axios.get('/sanctum/csrf-cookie')

        let response = await window.axios.post(
          '/api/v1/onboarding/database/config',
          databaseData
        )

        await window.axios.get('/sanctum/csrf-cookie')

        if (response.data.success) {
          await window.axios.post('/api/v1/onboarding/finish')

          this.$emit('next', 3)

          this.showNotification({
            type: 'success',
            message: this.$t('wizard.success.' + response.data.success),
          })

          return true
        } else if (response.data.error) {
          this.showNotification({
            type: 'error',
            message: this.$t('wizard.errors.' + response.data.error),
          })
        } else if (response.data.error_message) {
          this.showNotification({
            type: 'error',
            message: response.data.error_message,
          })
        }
      } catch (e) {
        this.showNotification({
          type: 'error',
          message: e.response.data.message,
        })
      } finally {
        this.isLoading = this.isFetching = false
      }
    },
  },
}
</script>
