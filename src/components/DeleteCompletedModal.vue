<!--
Nextcloud - Tasks

@author Raimund Schlüßler
@copyright 2019 Raimund Schlüßler <raimund.schluessler@mailbox.org>

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
	<div class="loadmore reactive">
		<span v-show="completedTasksCount" @click="openModal">
			{{ $t('tasks', 'Delete all completed tasks.') }}
		</span>
		<Modal v-if="modalOpen"
			:out-transition="true"
			@close="closeModal">
			<div class="emptycontent delete-completed">
				<p class="icon-delete" />
				<div v-if="completedTasksCount">
					<h3 class="delete-completed__header">
						{{ $n('tasks', 'This will delete {taskCount} completed task and its subtasks from calendar "{calendar}".', 'This will delete {taskCount} completed tasks and their subtasks from calendar "{calendar}".', initialCompletedRootTasksCount, {taskCount: initialCompletedRootTasksCount, calendar: calendar.displayName}, { sanitize: false, escape: false }) }}
					</h3>
					<button class="delete-completed__button icon-delete"
						type="button"
						@click="deleteCompletedTasks">
						{{ $t('tasks', 'Delete completed tasks.') }}
					</button>
				</div>
				<div v-else>
					<h3 class="delete-completed__header">
						{{ $t('tasks', 'Deleted all completed tasks from calendar "{calendar}".', { calendar: calendar.displayName }, undefined, { sanitize: false, escape: false }) }}
					</h3>
				</div>
				<div>
					<progress :max="initialCompletedTasksCount" :value="progress" class="delete-completed__progress" />
					<p class="delete-completed__tracker">
						<span>{{ percentage }} %</span>
						<span v-if="failed === 0">
							{{ $t('tasks', 'No errors') }}
						</span>
						<span v-else v-tooltip.auto="$t('tasks', 'Open your browser console for more details')">
							{{ $n('tasks', 'Could not delete {failedCount} task.', 'Could not delete {failedCount} tasks.', failed, { failedCount: failed }) }}
						</span>
					</p>
				</div>
			</div>
		</Modal>
	</div>
</template>

<script>
import Modal from '@nextcloud/vue/dist/Components/Modal'

import { mapGetters, mapActions } from 'vuex'

export default {
	components: {
		Modal,
	},
	props: {
		calendar: {
			type: Object,
			required: true,
		},
	},
	data() {
		return {
			modalOpen: false,
			initialCompletedTasksCount: 0,
			initialCompletedRootTasksCount: 0,
		}
	},
	computed: {
		loadedCompleted() {
			return this.calendar.loadedCompleted
		},
		tasks() {
			return this.closedRootTasks(this.calendar.tasks)
		},
		completedTasksCount() {
			const closedCount = function counter(tasks) {
				let i = tasks.length
				tasks.forEach((task) => {
					i += counter(Object.values(task.subTasks))
				})
				return i
			}
			return closedCount(this.tasks)
		},
		failed() {
			return 0
		},
		progress() {
			return this.initialCompletedTasksCount - this.completedTasksCount
		},
		percentage() {
			return this.initialCompletedTasksCount <= 0
				? 0
				: Math.floor(this.progress / this.initialCompletedTasksCount * 100)
		},
		...mapGetters({
			closedCount: 'getCalendarCountClosed',
			closedRootTasks: 'findClosedRootTasks',
		}),
	},
	methods: {
		...mapActions([
			'deleteTask',
		]),
		openModal() {
			this.modalOpen = true
			this.initialCompletedTasksCount = this.completedTasksCount
			// Number of completed root tasks
			this.initialCompletedRootTasksCount = this.tasks.length
		},
		closeModal() {
			this.modalOpen = false
		},
		deleteCompletedTasks() {
			this.tasks.map(
				(task) => this.deleteTask({ task, dav: true })
			)
		},
	},
}
</script>

<style lang="scss" scoped>
.loadmore {
	font-size: 11px;
	margin-top: 10px;
	text-align: center;
	height: 21px;

	span {
		color: var(--color-text-lighter);
		background-color: var(--color-main-background);
		border-radius: var(--border-radius-pill);
		padding: 3px 6px;

		&:hover {
			cursor: pointer;
			color: var(--color-main-text);
		}
	}
}

.delete-completed {
	margin: 50px;
	width: auto;
	min-width: 30vw;
	&__button {
		display: inline-block;
		padding: 10px;
		padding-left: 34px;
		background-position: 10px center;
		text-align: left;
		margin: 0;
		width: unset !important;
		height: unset !important;
		background-size: unset !important;
	}
	&__header {
		padding-top: 20px;
		max-width: 80%;
		margin: 12px auto;
	}
	&__progress {
		width: 80%;
		margin: auto;
	}
	&__tracker {
		display: flex;
		justify-content: space-between;
		width: 80%;
		margin: auto;
		padding-top: 10px;
	}
}
</style>
