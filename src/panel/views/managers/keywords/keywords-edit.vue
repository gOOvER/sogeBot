<template>
  <b-container fluid ref="window">
    <b-row>
      <b-col>
        <span class="title text-default mb-2">
          {{ translate('menu.manage') }}
          <small><i class="fas fa-angle-right"></i></small>
          {{ translate('menu.keywords') }}
          <template v-if="$route.params.id">
            <small><i class="fas fa-angle-right"></i></small>
            {{keyword}}
            <small>{{$route.params.id}}</small>
          </template>
        </span>
      </b-col>
    </b-row>

    <panel>
      <template v-slot:left>
        <button-with-icon class="btn-secondary btn-reverse" icon="caret-left" href="#/manage/keywords/list">{{translate('commons.back')}}</button-with-icon>
        <hold-button :if="$route.params.id || null" @trigger="del()" icon="trash" class="btn-danger">
          <template slot="title">{{translate('dialog.buttons.delete')}}</template>
          <template slot="onHoldTitle">{{translate('dialog.buttons.hold-to-delete')}}</template>
        </hold-button>
        <button-with-icon :class="[ enabled ? 'btn-success' : 'btn-danger' ]" class="btn-reverse" icon="power-off" @click="enabled = !enabled">
          {{ translate('dialog.buttons.' + (enabled? 'enabled' : 'disabled')) }}
        </button-with-icon>
      </template>
      <template v-slot:right>
        <b-alert show variant="info" v-if="state.pending" v-html="translate('dialog.changesPending')" class="mr-2 p-2 mb-0"></b-alert>
        <state-button @click="save()" text="saveChanges" :state="state.save" :invalid="!!$v.$invalid && !!$v.$dirty"/>
      </template>
    </panel>

    <loading v-if="state.loading === 1"/>
    <b-form v-else>
      <b-form-group
        :label="translate('systems.keywords.keyword.name')"
        label-for="keyword"
      >
        <b-form-input
          id="keyword"
          v-model="keyword"
          type="text"
          :placeholder="translate('systems.keywords.keyword.placeholder')"
          :state="$v.keyword.$invalid && $v.keyword.$dirty ? 'invalid' : null"
          @input="$v.keyword.$touch()"
        ></b-form-input>
        <b-form-invalid-feedback>{{ translate('dialog.errors.required') }}</b-form-invalid-feedback>
        <small class="form-text text-muted" v-html="translate('systems.keywords.keyword.help')"></small>
      </b-form-group>
      <b-form-group
        :label="translate('systems.keywords.response.name')"
        label-for="response"
      >
        <b-form-textarea
          id="response"
          v-model="response"
          :placeholder="translate('systems.keywords.response.placeholder')"
          :state="$v.response.$invalid && $v.response.$dirty ? 'invalid' : null"
          rows="8"
          @input="$v.response.$touch()"
        ></b-form-textarea>
        <b-form-invalid-feedback>{{ translate('dialog.errors.required') }}</b-form-invalid-feedback>
      </b-form-group>
    </b-form>
  </b-container>
</template>
<script lang="ts">
import { Vue, Component, Watch } from 'vue-property-decorator';

import { Validate } from 'vuelidate-property-decorators';
import { required } from 'vuelidate/lib/validators'

import uuid from 'uuid/v4';

import { KeywordInterface } from '../../../../bot/systems/keywords';

@Component({
  components: {
    'loading': () => import('../../../components/loading.vue'),
  },
})
export default class keywordsEdit extends Vue {
  socket = io('/systems/keywords', { query: "token=" + this.token });

  state: {
    loading: ButtonStates;
    save: ButtonStates;
    pending: boolean;
  } = {
    loading: ButtonStates.progress,
    save: ButtonStates.idle,
    pending: false,
  }

  id: string = uuid();
  @Validate({ required })
  keyword: string = '';
  @Validate({ required })
  response: string = '';
  enabled: boolean = true;

  @Watch('enabled')
  @Watch('keyword')
  @Watch('response')
  pending() {
    if (this.state.loading === ButtonStates.success) {
      this.state.pending = true;
    }
  }


  mounted() {
    if (this.$route.params.id) {
      this.socket.emit('findOne', { where: { id: this.$route.params.id } }, (err, data: KeywordInterface) => {
        if (err) {
          return console.error(err)
        }

        this.id = data.id;
        this.keyword = data.keyword;
        this.response = data.response;
        this.enabled = data.enabled;

        this.$nextTick(() => {
          this.state.loading = ButtonStates.success;
        });
      })
    } else {
      this.state.loading = ButtonStates.success;
    }
  }

  del() {
    this.socket.emit('delete', { where: { id: this.$route.params.id }}, (err, deleted) => {
      if (err) {
        return console.error(err);
      }
      this.$router.push({ name: 'KeywordsManagerList' })
    })
  }

  save() {
    this.$v.$touch();
    if (!this.$v.$invalid) {
      const keyword: KeywordInterface = {
        id: this.id,
        keyword: this.keyword,
        response: this.response,
        enabled: this.enabled,
      }
      this.state.save = ButtonStates.progress;

      this.socket.emit('update', { key: 'id', items: [keyword] }, (err, data) => {
        if (err) {
          this.state.save = ButtonStates.fail;
          return console.error(err);
        }

        this.state.save = ButtonStates.success;
        this.state.pending = false;
        this.$router.push({ name: 'KeywordsManagerEdit', params: { id: String(data.id) } })
        setTimeout(() => {
          this.state.save = ButtonStates.idle;
        }, 1000)
      });
    }
  }
}
</script>

<style scoped>
</style>