<!--
Nextcloud - Tasks

@author Christoph Wurst
@copyright 2021 Christoph Wurst <christoph@winzerhof-wurst.at>
@copyright 2021 Raimund Schlüßler <raimund.schluessler@mailbox.org>

This library is free software; you can redistribute it and/or
modify it under the terms of the GNU AFFERO GENERAL PUBLIC LICENSE
License as published by the Free Software Foundation; either
version 3 of the License, or any later version.

This library is distributed in the hope that it will be useful,
but WITHOUT ANY WARRANTY; without even the implied warranty of
MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See the
GNU AFFERO GENERAL PUBLIC LICENSE for more details.

You should have received a copy of the GNU Affero General Public
License along with this library. If not, see <http://www.gnu.org/licenses/>.

-->

<template>
	<AppNavigationItem :title="t('tasks', 'Trash bin')"
		:pinned="true"
		@click.prevent="onShow">
		<template #icon>
			<Delete :size="24" decorative />
		</template>
		<template #extra>
			<Modal v-if="showModal"
				@close="showModal = false">
				<div class="modal__content">
					<EmptyContent v-if="loading" icon="icon-loading">
						{{ t('tasks', 'Loading deleted calendars, tasks and events.') }}
					</EmptyContent>
					<EmptyContent v-else-if="!items.length">
						<template #icon>
							<Delete :size="24" decorative />
						</template>
						{{ t('tasks', 'You do not have any deleted calendars, tasks or events.') }}
					</EmptyContent>
					<template v-else>
						<h2>{{ t('tasks', 'Trash bin') }}</h2>
						<div class="table">
							<div class="table__header">
								{{ t('tasks', 'Name') }}
							</div>
							<div class="table__header table__header--deletedAt">
								{{ t('tasks', 'Deleted') }}
							</div>
							<div class="table__header">
								&nbsp;
							</div>
							<template v-for="item in items" class="row">
								<div :key="`${item.url}desc`" class="table__body">
									<div
										class="icon-bullet"
										:style="{ 'background-color': item.color }" />
									<div class="item-description">
										<div class="item-description__mainline">
											{{ item.name }}
										</div>
										<div v-if="item.subline" class="item-description__subline">
											{{ item.subline }}
										</div>
									</div>
								</div>
								<div :key="`${item.url}date`" class="table__body table__body--deletedAt">
									<Moment class="timestamp" :timestamp="item.deletedAt" />
								</div>
								<div :key="`${item.url}action`" class="table__body">
									<button @click="restore(item)">
										{{ t('tasks','Restore') }}
									</button>
									<Actions :force-menu="true">
										<ActionButton
											@click="onDeletePermanently(item)">
											<template #icon>
												<Delete :size="24" decorative />
											</template>
											{{ t('tasks','Delete permanently') }}
										</ActionButton>
									</Actions>
								</div>
							</template>
						</div>
						<p v-if="retentionDuration" class="footer">
							{{ n('calendar', 'Elements in the trash bin are deleted after {numDays} day', 'Elements in the trash bin are deleted after {numDays} days', retentionDuration, { numDays: retentionDuration }) }}
						</p>
					</template>
				</div>
			</Modal>
		</template>
	</AppNavigationItem>
</template>

<script>
import Moment from './Moment'
import { uidToHexColor } from '../../utils/color'
import logger from '../../utils/logger'

import { showError } from '@nextcloud/dialogs'
import { translate as t } from '@nextcloud/l10n'
import moment from '@nextcloud/moment'
import AppNavigationItem from '@nextcloud/vue/dist/Components/AppNavigationItem'
import Actions from '@nextcloud/vue/dist/Components/Actions'
import ActionButton from '@nextcloud/vue/dist/Components/ActionButton'
import EmptyContent from '@nextcloud/vue/dist/Components/EmptyContent'
import Modal from '@nextcloud/vue/dist/Components/Modal'

import Delete from 'vue-material-design-icons/Delete.vue'

import { mapGetters } from 'vuex'

export default {
	name: 'Trashbin',
	components: {
		AppNavigationItem,
		Delete,
		EmptyContent,
		Modal,
		Moment,
		Actions,
		ActionButton,
	},
	data() {
		return {
			showModal: false,
			loading: true,
		}
	},
	computed: {
		...mapGetters({
			trashBin: 'trashBin',
			calendars: 'sortedDeletedCalendars',
			objects: 'deletedCalendarObjects',
		}),
		items() {
			const formattedCalendars = this.calendars.map(calendar => ({
				calendar,
				type: 'calendar',
				key: calendar.url,
				name: calendar.displayname,
				url: calendar._url,
				deletedAt: calendar._props['{http://nextcloud.com/ns}deleted-at'],
				color: calendar.color ?? uidToHexColor(calendar.displayname),
			}))
			const formattedCalendarObjects = this.objects.map(vobject => {
				let eventSummary = t('tasks', 'Untitled item')
				try {
					eventSummary = vobject?.calendarComponent.getComponentIterator().next().value?.title
				} catch (e) {
					// ignore
				}
				let subline = vobject.calendar.displayName
				if (vobject.isEvent) {
					const event = vobject?.calendarComponent.getFirstComponent('VEVENT')
					if (event?.startDate.jsDate && event?.isAllDay()) {
						subline += ' · ' + moment(event.startDate.jsDate).format('LL')
					} else if (event?.startDate.jsDate) {
						subline += ' · ' + moment(event?.startDate.jsDate).format('LLL')
					}
				}
				const color = vobject.calendarComponent.getComponentIterator().next().value?.color
						?? vobject.calendar.color
						?? uidToHexColor(vobject.calendar.displayName)
				return {
					vobject,
					type: 'object',
					key: vobject.id,
					name: eventSummary,
					subline,
					url: vobject.uri,
					deletedAt: vobject.dav._props['{http://nextcloud.com/ns}deleted-at'],
					color,
				}
			})

			return formattedCalendars.concat(formattedCalendarObjects).sort((item1, item2) => item2.deletedAt - item1.deletedAt)

		},
		retentionDuration() {
			return Math.ceil(
				this.trashBin.retentionDuration / (60 * 60 * 24)
			)
		},
	},
	methods: {
		async onShow() {
			this.showModal = true

			this.loading = true
			try {
				await Promise.all([
					this.$store.dispatch('loadDeletedCalendars'),
					this.$store.dispatch('loadDeletedCalendarObjects'),
				])

				logger.debug('deleted calendars and objects loaded', {
					calendars: this.calendars,
					objects: this.objects,
				})
			} catch (error) {
				logger.error('could not load deleted calendars and objects', {
					error,
				})

				showError(t('tasks', 'Could not load deleted calendars and objects'))
			}
			this.loading = false
		},
		async onDeletePermanently(item) {
			logger.debug('deleting ' + item.url + ' permanently', item)
			try {
				switch (item.type) {
				case 'calendar':
					await this.$store.dispatch('deleteCalendarPermanently', { calendar: item.calendar })
					break
				case 'object':
					await this.$store.dispatch('deleteCalendarObjectPermanently', { vobject: item.vobject })
					break
				}
			} catch (error) {
				logger.error('could not restore ' + item.url, { error })

				showError(t('tasks', 'Could not restore calendar or event'))
			}
		},
		async restore(item) {
			logger.debug('restoring ' + item.url, item)
			try {
				switch (item.type) {
				case 'calendar': {
					await this.$store.dispatch('restoreCalendar', { calendar: item.calendar })
					const { calendars } = await this.$store.dispatch('getCalendarsAndTrashBin')
					// Load the tasks of the restored calendar
					const calendar = calendars.find(cal => cal.url === item.calendar.url)
					if (calendar) {
						await this.$store.dispatch('getTasksFromCalendar', { calendar, completed: false, related: null })
					}
					break
				}
				case 'object':
					await this.$store.dispatch('restoreCalendarObject', { vobject: item.vobject })
					break
				}
			} catch (error) {
				logger.error('could not restore ' + item.url, { error })

				showError(t('tasks', 'Could not restore calendar or event'))
			}
		},
	},
}
</script>

<style lang="scss" scoped>
::v-deep .modal-container {
	height: 80%;
}

.modal__content {
	display: flex;
	flex-direction: column;
	margin: 2vw;
	height: calc(100% - 4vw);
	max-height: calc(100% - 4vw);
}

.table {
	display: grid;
	grid-template-columns: minmax(200px, 1fr) max-content max-content;
	overflow: scroll;
	margin-bottom: auto;

	&__header {
		position: sticky;
		top: 0;
		background-color: var(--color-main-background-translucent);
		z-index: 1;
	}

	&__header,
	&__body {
		&--deletedAt {
			justify-content: right;
		}
	}

	& > div {
		display: flex;
		align-items: center;
		padding: 4px;
	}
}

.item-description {
	overflow: hidden;

	&__mainline,
	&__subline {
		overflow: hidden;
		white-space: nowrap;
		text-overflow: ellipsis;
	}

	&__subline {
		color: var(--color-text-maxcontrast);
	}
}

.footer {
	color: var(--color-text-lighter);
	text-align: center;
	font-size: small;
	margin-top: 16px;
}

.icon-bullet {
	display: inline-block;
	vertical-align: middle;
	width: 14px;
	height: 14px;
	margin-right: 14px;
	border: none;
	border-radius: 50%;
	flex-shrink: 0;
}
</style>
