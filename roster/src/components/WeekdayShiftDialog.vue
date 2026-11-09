<template>
	<Dialog v-model="open" :options="{ title: `Change every ${props.weekdayName}`, size: 'xl' }">
		<template #body-content>
			<div class="space-y-5">
				<div class="rounded-md bg-gray-50 p-3 text-sm">
					<div class="text-xs text-gray-600 mb-1">
						{{ props.employees.length === 1 ? "Employee" : `Employees (${props.employees.length})` }}
					</div>
					<div class="font-medium">{{ employeeText }}</div>
				</div>

				<p class="text-sm text-gray-600">
					Gives {{ props.employees.length === 1 ? "this employee" : "these employees" }} the
					shift below on every {{ props.weekdayName }}. Only future days change: days with a check-in or
					attendance, and days that already have this shift, are left alone.
				</p>

				<Link doctype="Shift Type" label="Shift Type" v-model="form.shift_type" />

				<FormControl
					v-if="allowMultiple"
					type="select"
					label="If the day already has a shift"
					v-model="form.mode"
					:options="[
						{ label: 'Replace it', value: 'replace' },
						{ label: 'Add alongside it (a different shift)', value: 'add' },
					]"
					:description="
						form.mode === 'add'
							? 'Needs Allow Multiple Shift Assignments in HR Settings and shift times that do not overlap. The same shift is never added twice.'
							: ''
					"
				/>

				<div class="grid grid-cols-2 gap-4">
					<FormControl type="date" label="From Date" v-model="form.from_date" :min="today" />
					<FormControl
						type="date"
						label="To Date (optional)"
						v-model="form.to_date"
						description="Blank = until the last day already rostered"
					/>
				</div>

				<div v-if="result" class="rounded-md bg-gray-50 p-3 text-sm space-y-1">
					<div v-if="'removed' in result">
						<b>{{ result.removed }}</b> removed, <b>{{ result.no_shift }}</b> had no shift,
						<b>{{ result.skipped.length }}</b> skipped.
					</div>
					<div v-else>
						<b>{{ result.changed }}</b> changed, <b>{{ result.unchanged }}</b> already had
						this shift, <b>{{ result.skipped.length }}</b> skipped.
					</div>
					<ul v-if="result.skipped.length" class="max-h-32 overflow-auto text-xs text-gray-600">
						<li v-for="(s, i) in result.skipped" :key="i">
							{{ s.employee }} · {{ s.date }} — {{ s.reason }}
						</li>
					</ul>
				</div>
			</div>
		</template>
		<template #actions>
			<div class="flex justify-end space-x-3">
				<Button
					variant="subtle"
					theme="red"
					:loading="remove.loading"
					:disabled="!props.employees.length"
					@click="confirmDelete ? remove.submit() : (confirmDelete = true)"
				>
					{{ confirmDelete ? `Confirm: remove every ${props.weekdayName}` : "Delete" }}
				</Button>
				<Button
					variant="solid"
					class="w-28"
					:loading="apply.loading"
					:disabled="!form.shift_type || !props.employees.length"
					@click="apply.submit()"
				>
					Apply
				</Button>
			</div>
		</template>
	</Dialog>
</template>

<script setup lang="ts">
import { computed, reactive, ref, watch } from "vue";
import { Dialog, FormControl, createResource } from "frappe-ui";

import Link from "./Link.vue";
import { dayjs, raiseToast } from "../utils";

type Result = {
	changed: number;
	unchanged: number;
	skipped: { employee: string; date: string; reason: string }[];
};

type RemoveResult = {
	removed: number;
	no_shift: number;
	skipped: { employee: string; date: string; reason: string }[];
};

const props = defineProps<{
	modelValue: boolean;
	weekday: number; // 0 = Monday ... 6 = Sunday
	weekdayName: string;
	employees: string[];
	employeeLabels?: string[];
	// The Roster's custom From/To range, when one is set
	defaultFrom?: string;
	defaultTo?: string;
}>();

const emit = defineEmits<{
	(e: "update:modelValue", value: boolean): void;
	(e: "applied"): void;
}>();

const today = dayjs().format("YYYY-MM-DD");
const open = computed({
	get: () => props.modelValue,
	set: (value) => emit("update:modelValue", value),
});
const MAX_NAMES = 6;
const employeeText = computed(() => {
	const names = props.employeeLabels?.length ? props.employeeLabels : props.employees;
	return names.length > MAX_NAMES
		? `${names.slice(0, MAX_NAMES).join(", ")} and ${names.length - MAX_NAMES} more`
		: names.join(", ");
});
const form = reactive({ shift_type: "", from_date: today, to_date: "", mode: "replace" });
const result = ref<Result | RemoveResult>();
const confirmDelete = ref(false);

watch(
	() => props.modelValue,
	(isOpen) => {
		if (isOpen) {
			options.fetch();
			Object.assign(form, {
				mode: "replace",
				shift_type: "",
				from_date: props.defaultFrom || today,
				to_date: props.defaultTo || "",
			});
			result.value = undefined;
			confirmDelete.value = false;
		}
	},
);

const remove = createResource({
	url: "et_adms.custom.weekday_shift.remove_weekday_shift",
	makeParams: () => ({
		employees: props.employees,
		weekday: props.weekday,
		from_date: form.from_date || today,
		to_date: form.to_date || null,
	}),
	onSuccess(data: RemoveResult) {
		result.value = data;
		confirmDelete.value = false;
		raiseToast("success", `${props.weekdayName}: ${data.removed} shift(s) removed`);
		emit("applied");
	},
	onError(error: { messages: string[] }) {
		confirmDelete.value = false;
		raiseToast("error", error.messages[0]);
	},
});

// "Add alongside" only exists when HR Settings allows several shifts on one day
const options = createResource({
	url: "et_adms.custom.weekday_shift.get_weekday_shift_options",
	auto: true,
});
const allowMultiple = computed(() => !!options.data?.allow_multiple);

const apply = createResource({
	url: "et_adms.custom.weekday_shift.apply_weekday_shift",
	makeParams: () => ({
		employees: props.employees,
		weekday: props.weekday,
		shift_type: form.shift_type,
		from_date: form.from_date || today,
		to_date: form.to_date || null,
		mode: allowMultiple.value ? form.mode : "replace",
	}),
	onSuccess(data: Result) {
		result.value = data;
		raiseToast("success", `${props.weekdayName}: ${data.changed} shift(s) changed`);
		emit("applied");
	},
	onError(error: { messages: string[] }) {
		raiseToast("error", error.messages[0]);
	},
});
</script>
