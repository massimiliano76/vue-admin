<script>
	import http from 'src/http';
	import { asFormData, parsePlaceholders, httpErrorModalData, formatFieldErrors } from 'src/utils';

	function defaultFieldValue(field) {
		if (field.type === 'boolean' || field.type === 'bool' || field.type === 'checkbox') return false;
		if (field.options && field.multiple) return [];
		return null;
	}

	export default {
		props: {
			entity: {
				type: String,
				default() {
					return this.$route.path.split('/')[2];
				},
			},
			id: null,
		},
		data() {
			return {
				error: null,
				fieldErrors: {},
				loading: false,
				item: null,
				initialState: null,
				actionsOpen: false,
			};
		},
		computed: {
			creating() {
				return !this.id || this.id === 'new';
			},
			apiPath() {
				return 'entity/' + this.entity;
			},
			basePath() {
				return '/' + this.apiPath;
			},
			meta() {
				return this.$store.getters.entitiesData[this.entity];
			},
			title() {
				return this.meta && this.meta.title || this.creating && this.$t('creating') || this.$t('editing');
			},
			subtitle() {
				if (this.creating) return this.meta.create_title || this.$t('newElement');
				if (this.item) {
					if (this.meta.item_title) return parsePlaceholders(this.meta.item_title, this.item);
					return (this.initialState.title || this.initialState.name || this.initialState.label) + ' [' + this.id + ']';
				}
				return this.$t('loading') + '...';
			},
			itemApiPath() {
				if (this.creating) return this.apiPath;
				return this.apiPath + '/' + this.id;
			},
			fields() {
				return this.meta.item_fields.filter(field => field.type !== 'hidden');
			},
			hasErrors() {
				return this.error || Object.keys(this.fieldErrors).length;
			},
		},
		created() {
			this.update();
		},
		beforeDestroy() {
			window.removeEventListener('click', this.closeActions);
		},
		methods: {
			setNewItem() {
				const item = {};
				for (let field of this.fields) {
					item[field.name] = field.hasOwnProperty('default') ? field.default : defaultFieldValue(field);
				}
				this.item = item;
				this.initialState = JSON.parse(JSON.stringify(item));
			},
			itemPathFor(item) {
				return this.basePath + '/item/' + item[this.meta && this.meta.primary || 'id'];
			},
			clearErrors() {
				this.error = '';
				this.fieldErrors = {};
			},
			permitted(action) {
				return !this.meta.permissions || this.meta.permissions[action] !== false;
			},
			postData() {
				for (let field of Object.keys(this.item)) {
					if (this.item[field] instanceof File || this.item[field] instanceof FileList) {
						return asFormData(this.item);
					}
				}
				return this.item;
			},
			update() {
				if (this.creating) {
					this.setNewItem();
				}
				else {
					this.loading = true;
					http.get(this.itemApiPath)
						.then(res => {
							this.item = res.data;
							this.initialState = JSON.parse(JSON.stringify(this.item));
						})
						.catch(err => {
							this.error = err.response.status;
						})
						.then(() => {
							this.loading = false;
						});
				}
			},
			destroy() {
				this.$modal.open('confirm', {
					title: this.$t('deleteElement') + '?',
					text: this.initialState.title || this.initialState.name,
				}, 'modal-sm').then(result => {
					if (result === true) {
						this.loading = true;
						http.delete(this.itemApiPath)
							.then(() => {
								this.back();
							})
							.catch(err => {
								this.$modal.open('error', {
									...httpErrorModalData(err),
									text: this.$t('errors.deleteElement'),
								});
								this.loading = false;
							});
					}
				});
			},
			submit(cb) {
				this.clearErrors();
				this.loading = true;
				http.post(this.itemApiPath, this.postData())
					.then(res => {
						this.item = res.data;
						this.initialState = JSON.parse(JSON.stringify(this.item));
						if (cb) cb();
					})
					.catch(err => {
						if (err.response && err.response.status === 422)
							this.fieldErrors = formatFieldErrors(err.response.data.errors);
						this.$modal.open('error', {
							...httpErrorModalData(err),
							title: this.$t('errors.saveElement'),
						});
					})
					.then(() => {
						this.loading = false;
					});
			},
			save() {
				this.submit(() => {
					const path = this.itemPathFor(this.item);
					if (path !== this.$route.fullPath) this.$router.replace(this.itemPathFor(this.item));
					this.$notify(this.$t('savedMessage'), { class: 'alert-success' });
				});
			},
			reset() {
				this.clearErrors();
				this.item = JSON.parse(JSON.stringify(this.initialState));
			},
			back() {
				this.$router.back();
			},
			saveAndBack() {
				this.submit(() => {
					this.back();
				});
			},
			fieldHidden(name) {
				return this.hiddenFields && this.hiddenFields.indexOf(name) !== -1;
			},
			fieldDisabled(name) {
				return this.disabledFields && this.disabledFields.indexOf(name) !== -1;
			},
			closeActions() {
				this.toggleActions(false);
			},
			toggleActions(to) {
				if (to === undefined) to = !this.actionsOpen;
				this.actionsOpen = to;
				this.$nextTick(() => {
					window[(to ? 'add' : 'remove') + 'EventListener']('click', this.closeActions);
				});
			},
		},
		beforeRouteUpdate(to, old, next) {
			if (to.path !== old.path) {
				this.items = null;
				this.total = 0;
				this.lastPage = 1;
			}
			next();
			this.$nextTick(() => {
				this.update();
			});
		},
	};
</script>
<template lang="pug">
	page.entity-item-page(:class="entity + '-item-page'")
		span(slot="title")!='{{ title }} '
			small {{ subtitle }}
		template(slot="breadcrumbs")
			li: router-link(:to="basePath") {{ meta && meta.title || $t('elementList') }}
			li.active: span {{ subtitle }}
		.box(v-if="fields")
			spinner(v-if="loading")
			form.box-body(@submit.prevent="saveAndBack()" v-if="item")
				field(v-for="field in fields" :key="field.name"
					v-show="!fieldHidden(field.name)"
					v-model="item[field.name]"
					:disabled="fieldDisabled(field.name)"
					:object="item"
					:errors="fieldErrors[field.name]"
					v-bind="field")
				.btn-group
					button.btn.btn-success {{ $t('saveAndReturn') }}
					button.btn.btn-success(@click.prevent="save()") {{ $t('save') }}
				!=' '
				.btn-group.dropup(:class="{ open: actionsOpen }")
					button.btn.btn-default.dropdown-toggle(type="button" @click.stop="toggleActions()"): span.caret
					ul.dropdown-menu
						li: a(type="button" @click="toggleActions(), reset()") {{ $t('reset') }}
						li(v-if="!creating && permitted('destroy')"): a(type="button" @click="toggleActions(), destroy()"): span.text-danger {{ $t('delete') }}
						li: a(@click="toggleActions(), back()") {{ $t('back') }}
		.callout.callout-warning(v-else)
			h4 {{ $t('noMetaMessage') }}
			code.
				ModelHandler::setItemFields() or Model::setFillable()
</template>
<style lang="stylus">
	.entity-item-page
		.box, form.box-body
			overflow visible
</style>