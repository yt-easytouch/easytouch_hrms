<template>
	<div class="flex items-center">
		<!-- Month Change -->
		<div v-if="!props.rangeFrom || !props.rangeTo" class="flex items-center bg-gray-50 rounded-md space-x-0.5">
			<Button icon="chevron-left" variant="ghost" @click="emit('addToMonth', -1)" />
			<span class="w-32 text-center font-medium text-base">
				{{ props.firstOfMonth.format("MMMM") }}, {{ firstOfMonth.format("YYYY") }}
			</span>
			<Button icon="chevron-right" variant="ghost" @click="emit('addToMonth', 1)" />
		</div>

		<!-- Custom date range (replaces the month while both are set) -->
		<div v-if="props.showRange" class="ml-2.5 flex items-center space-x-1.5" title="Custom date range">
			<span class="text-xs text-gray-500 uppercase tracking-wide">Range</span>
			<div class="w-36">
				<FormControl
					type="date"
					:modelValue="props.rangeFrom"
					placeholder="From"
					@update:modelValue="(v: string) => emit('updateRange', { from: v || '', to: props.rangeTo })"
				/>
			</div>
			<span class="text-gray-500 text-sm">→</span>
			<div class="w-36">
				<FormControl
					type="date"
					:modelValue="props.rangeTo"
					placeholder="To"
					:min="props.rangeFrom"
					@update:modelValue="(v: string) => emit('updateRange', { from: props.rangeFrom, to: v || '' })"
				/>
			</div>
			<Button
				v-if="props.rangeFrom || props.rangeTo"
				icon="x"
				variant="ghost"
				title="Back to month"
				@click="emit('updateRange', { from: '', to: '' })"
			/>
		</div>

		<!-- Day filter: one weekday (e.g. every Friday) or a single date -->
		<div v-if="props.showDayFilter" class="ml-2.5 flex items-center space-x-1.5" title="Which days to show">
			<span class="text-xs text-gray-500 uppercase tracking-wide">Show</span>
			<div class="w-44">
			<FormControl
				type="select"
				:options="dayOptions"
				:modelValue="props.dayFilter"
				@update:modelValue="(v: string) => emit('updateDay', v)"
			/>
			</div>
		</div>

		<!-- Filters -->
		<div class="ml-auto space-x-2.5 flex">
			<div v-for="[key, value] of Object.entries(filters)" :key="key" class="w-40">
				<FormControl
					type="autocomplete"
					:placeholder="toTitleCase(key)"
					:options="value.options"
					v-model="value.model"
					:disabled="!value.options.length"
				/>
			</div>
			<Button icon="x" @click="Object.values(filters).forEach((d) => (d.model = null))" />
		</div>
	</div>
</template>

<script setup lang="ts">
import { reactive, watch } from "vue";
import { FormControl, createResource, createListResource } from "frappe-ui";
import { Dayjs } from "dayjs";

import { raiseToast } from "../utils";

export type FilterField =
	| "company"
	| "department"
	| "branch"
	| "designation"
	| "shift_type"
	| "shift_location";

const props = defineProps<{
	firstOfMonth: Dayjs;
	dayFilter: string;
	rangeFrom: string;
	rangeTo: string;
	showRange?: boolean;
	showDayFilter?: boolean;
}>();

const emit = defineEmits<{
	(e: "addToMonth", change: number): void;
	(e: "updateDay", value: string): void;
	(e: "updateRange", value: { from: string; to: string }): void;
	(e: "updateFilters", newFilters: { [K in FilterField]: string }): void;
}>();

const filters: {
	[K in FilterField]: {
		options: string[];
		model?: { value: string } | null;
	};
} = reactive({
	company: { options: [], model: null },
	department: { options: [], model: null },
	branch: { options: [], model: null },
	designation: { options: [], model: null },
	shift_type: { options: [], model: null },
	shift_location: { options: [], model: null },
});

watch(
	() => filters.company.model,
	(val) => {
		if (val?.value) getFilterOptions("department", { company: val.value });
		else {
			filters.department.model = null;
			filters.department.options = [];
		}
	},
);

watch(filters, (val) => {
	const newFilters = {
		company: val.company.model?.value || "",
		department: val.department.model?.value || "",
		branch: val.branch.model?.value || "",
		designation: val.designation.model?.value || "",
		shift_type: val.shift_type.model?.value || "",
		shift_location: val.shift_location.model?.value || "",
	};
	emit("updateFilters", newFilters);
});

const WEEKDAYS = ["Sunday", "Monday", "Tuesday", "Wednesday", "Thursday", "Friday", "Saturday"];

const dayOptions = computed(() => [
	{ label: "All days", value: "" },
	{ label: "Weekly pattern (Mon–Sun)", value: "week" },
	...WEEKDAYS.map((label, idx) => ({ label: `Every ${label}`, value: `dow:${idx}` })),
	...Array.from({ length: props.firstOfMonth.daysInMonth() }, (_, i) => {
		const date = props.firstOfMonth.date(i + 1);
		return { label: date.format("ddd DD MMM"), value: `date:${date.format("YYYY-MM-DD")}` };
	}),
]);

const toTitleCase = (str: string) =>
	str
		.split("_")
		.map((s) => s.charAt(0).toUpperCase() + s.slice(1))
		.join(" ");

// RESOURCES

const defaultCompany = createResource({
	url: "hrms.api.roster.get_default_company",
	auto: true,
	onSuccess: () => {
		["company", "branch", "designation", "shift_type", "shift_location"].forEach((field) =>
			getFilterOptions(field as FilterField),
		);
	},
});

const getFilterOptions = (field: FilterField, listFilters: { company?: string } = {}) => {
	createListResource({
		doctype: toTitleCase(field),
		fields: ["name"],
		filters: listFilters,
		pageLength: 100,
		auto: true,
		onSuccess: (data: { name: string }[]) => {
			const value = field === "company" ? defaultCompany.data : "";
			filters[field].model = { value };
			filters[field].options = data.map((item) => item.name);
		},
		onError(error: { messages: string[] }) {
			raiseToast("error", error.messages[0]);
		},
	});
};
</script>
