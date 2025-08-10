<script setup>
import { CountryService } from '@/service/CountryService';
import { NodeService } from '@/service/NodeService';
import { onMounted, ref, computed } from 'vue';
import { useToast } from 'primevue/usetoast';
import AppMenuItem from '@/layout/AppMenuItem.vue';
import reportingMonthsData from '@/mocks/reporting-months.json';

const autoValue = ref(null);
const autoFilteredValue = ref([]);
const toast = useToast();
const sliderValueRef = ref(50);

const sliderValue = computed({
    get() {
        return sliderValueRef.value;
    },
    set(newValue) {
        sliderValueRef.value = newValue;

        // Сохраняем изменения в модель при изменении слайдера
        if (selectedMonth.value) {
            selectedMonth.value.slider_value = newValue;

            const monthIndex = model.value.findIndex((m) => m.id === selectedMonth.value.id);
            if (monthIndex !== -1) {
                model.value[monthIndex].slider_value = newValue;
            }

            const filteredIndex = filteredModel.value.findIndex((m) => m.id === selectedMonth.value.id);
            if (filteredIndex !== -1) {
                filteredModel.value[filteredIndex].slider_value = newValue;
            }
        }
    }
});

const knobValue = ref(50);
const inputGroupValue = ref(false);

const model = ref([]);
const selectedMonth = ref(null);
const filterText = ref('');
const filteredModel = ref([]);
const showCreateModal = ref(false);
const isEditMode = ref(false);
const editingMonth = ref(null);
const newMonth = ref({
    year: new Date().getFullYear(),
    month: 1
});

const yearError = ref('');
const shakeAnimation = ref(false);

const monthOptions = ref([
    { label: 'Январь', value: 1 },
    { label: 'Февраль', value: 2 },
    { label: 'Март', value: 3 },
    { label: 'Апрель', value: 4 },
    { label: 'Май', value: 5 },
    { label: 'Июнь', value: 6 },
    { label: 'Июль', value: 7 },
    { label: 'Август', value: 8 },
    { label: 'Сентябрь', value: 9 },
    { label: 'Октябрь', value: 10 },
    { label: 'Ноябрь', value: 11 },
    { label: 'Декабрь', value: 12 }
]);

// Пропорциональное преобразование рейтинга
const ratingValue = computed({
    get() {
        return sliderValue.value / 20; // 100 / 5 = 20
    },
    set(newValue) {
        sliderValue.value = newValue * 20; // 5 звезд * 20 = 100
    }
});

onMounted(() => {
    CountryService.getCountries().then((data) => (autoValue.value = data));

    // Загружаем данные месяцев из JSON файла
    model.value = reportingMonthsData.results.map((month) => ({
        label: month.name,
        value: month.value,
        id: month.id,
        year: month.year,
        month: month.month,
        is_active: month.is_active,
        slider_value: month.slider_value || 50
    }));

    // Инициализируем отфильтрованный список
    filteredModel.value = [...model.value];
});

function searchCountry(event) {
    setTimeout(() => {
        if (!event.query.trim().length) {
            autoFilteredValue.value = [...autoValue.value];
        } else {
            autoFilteredValue.value = autoValue.value.filter((country) => {
                return country.name.toLowerCase().startsWith(event.query.toLowerCase());
            });
        }
    }, 250);
}

function editMonth(month) {
    isEditMode.value = true;
    editingMonth.value = month;
    newMonth.value = {
        year: month.year,
        month: month.month
    };
    // Проверяем валидность данных при открытии для редактирования
    validateYear();
    showCreateModal.value = true;
}

function selectMonth(month) {
    selectedMonth.value = month;
    // Загружаем значение слайдера из выбранного месяца
    sliderValueRef.value = month.slider_value || 50;
}

function filterMonths() {
    if (!filterText.value.trim()) {
        filteredModel.value = [...model.value];
    } else {
        filteredModel.value = model.value.filter((month) => month.label.toLowerCase().includes(filterText.value.toLowerCase()));
    }
}

function openCreateModal() {
    isEditMode.value = false;
    editingMonth.value = null;
    showCreateModal.value = true;
    // Сброс формы
    newMonth.value = {
        year: new Date().getFullYear(),
        month: 1
    };
}

function validateYear() {
    if (newMonth.value.year > 2100) {
        yearError.value = 'Год не может быть больше 2100';
        triggerShakeAnimation();
    } else if (newMonth.value.year < 2000) {
        yearError.value = 'Год не может быть меньше 2000';
        triggerShakeAnimation();
    } else {
        yearError.value = '';
        shakeAnimation.value = false;
    }
}

function triggerShakeAnimation() {
    shakeAnimation.value = false;
    // nextTick чтобы убедиться, что класс был удален перед добавлением
    setTimeout(() => {
        shakeAnimation.value = true;
    }, 10);
}

function closeCreateModal() {
    showCreateModal.value = false;
    isEditMode.value = false;
    editingMonth.value = null;
    yearError.value = '';
    shakeAnimation.value = false;
}

function createMonth() {
    validateYear();
    if (yearError.value) {
        return;
    }

    const monthName = monthOptions.value.find((m) => m.value === newMonth.value.month)?.label;
    const monthLabel = `${monthName} ${newMonth.value.year}`;
    const monthValue = `${newMonth.value.year}-${String(newMonth.value.month).padStart(2, '0')}`;

    if (isEditMode.value && editingMonth.value) {
        // Редактируем существующий месяц
        const monthIndex = model.value.findIndex((m) => m.id === editingMonth.value.id);
        if (monthIndex !== -1) {
            model.value[monthIndex] = {
                ...editingMonth.value,
                label: monthLabel,
                value: monthValue,
                year: newMonth.value.year,
                month: newMonth.value.month
            };
        }
    } else {
        // Создаем новый месяц
        const newId = Math.max(...model.value.map((m) => m.id)) + 1;

        const monthToAdd = {
            id: newId,
            label: monthLabel,
            value: monthValue,
            year: newMonth.value.year,
            month: newMonth.value.month,
            is_active: true,
            slider_value: 50
        };

        model.value.push(monthToAdd);
    }

    // Обновляем отфильтрованный список
    filterMonths();

    toast.add({
        severity: 'success',
        summary: isEditMode.value ? 'Месяц обновлен' : 'Месяц создан',
        detail: isEditMode.value ? `Месяц "${monthLabel}" успешно обновлен` : `Месяц "${monthLabel}" успешно создан`,
        life: 3000
    });

    closeCreateModal();
}
</script>

<template>
    <Fluid class="flex flex-col md:flex-row gap-8">
        <div class="md:w-1/3">
            <div class="card flex flex-col gap-2 items-start">
                <h5 class="font-semibold m-0 p-0">Отчетные месяцы</h5>
                <Button label="Создать" icon="pi pi-plus" text severity="primary" @click="openCreateModal" class="!w-auto" />
                <InputGroup>
                    <InputText v-model="filterText" placeholder="Введите название месяца..." @input="filterMonths" />
                    <Button icon="pi pi-search" severity="secondary" @click="filterMonths" />
                </InputGroup>
                <ul class="layout-menu">
                    <template v-for="(item, i) in filteredModel" :key="item.id">
                        <li class="layout-menuitem sdd py-1">
                            <div class="layout-menuitem-root-text">
                                <ButtonGroup>
                                    <Button icon="pi pi-pencil" :severity="selectedMonth?.id === item.id ? 'primary' : 'secondary'" @click="editMonth(item)" />
                                    <Button :label="item.label" :severity="selectedMonth?.id === item.id ? 'primary' : 'secondary'" text class="flex-1 justify-start" @click="selectMonth(item)" />
                                </ButtonGroup>
                            </div>
                        </li>
                    </template>
                </ul>
            </div>
        </div>
        <div class="md:w-2/3">
            <div class="card flex flex-col gap-4">
                <div class="content-area">
                    <div class="selected-month-info" v-if="selectedMonth">
                        <h4 class="font-semibold p-0">Выбранный месяц: {{ selectedMonth.name }}</h4>
                        <p><strong>Год:</strong> {{ selectedMonth.year }}</p>
                        <p><strong>Месяц:</strong> {{ monthOptions.find((m) => m.value === selectedMonth.month)?.label }}</p>
                        <div class="flex flex-col gap-4">
                            <div class="font-semibold text-xl">Slider</div>
                            <InputText v-model.number="sliderValue" :key="`input-${selectedMonth.id}`" />
                            <Slider v-model="sliderValue" :key="`slider-${selectedMonth.id}`" />
                        </div>

                        <div class="flex flex-row mt-6 mb-4">
                            <div class="flex flex-col gap-4 w-1/2">
                                <div class="font-semibold text-xl">Rating</div>
                                <Rating v-model="ratingValue" :key="`rating-${selectedMonth.id}`" />
                            </div>
                        </div>
                        <div class="font-semibold text-xl">Knob</div>
                        <Knob v-model="sliderValue" :step="10" :min="0" :max="100" valueTemplate="{value}%" :key="`knob-${selectedMonth.id}`" />

                        <div class="mt-6">
                            <Button label="Редактировать" text icon="pi pi-pencil" severity="primary" @click="editMonth(selectedMonth)" />
                        </div>
                    </div>

                    <div v-else>
                        <Message severity="secondary">Выберите месяц из списка слева</Message>
                    </div>
                </div>
            </div>
        </div>
    </Fluid>

    <!-- Модальное окно для создания нового месяца -->
    <Dialog v-model:visible="showCreateModal" modal :header="isEditMode ? 'Изменить отчетный месяц' : 'Создать новый отчетный месяц'" :style="{ width: '450px' }" :closable="true" @hide="closeCreateModal">
        <div class="flex flex-col gap-4">
            <div class="flex flex-col gap-2">
                <label for="monthYear" class="font-semibold">Год</label>
                <InputNumber id="monthYear" v-model="newMonth.year" :use-grouping="false" :invalid="!!yearError" :class="['w-full', { 'shake-animation': shakeAnimation }]" placeholder="Введите год" @update:modelValue="validateYear" />
                <small v-if="yearError" class="text-red-500 text-sm">{{ yearError }}</small>
            </div>

            <div class="flex flex-col gap-2">
                <label for="monthSelect" class="font-semibold">Месяц</label>
                <Select id="monthSelect" v-model="newMonth.month" :options="monthOptions" optionLabel="label" optionValue="value" placeholder="Выберите месяц" class="w-full" />
            </div>
        </div>

        <template #footer>
            <div class="flex justify-end gap-2">
                <Button label="Отмена" severity="secondary" @click="closeCreateModal" />
                <Button :label="isEditMode ? 'Сохранить' : 'Создать'" severity="primary" @click="createMonth" :disabled="!newMonth.year || !newMonth.month" />
            </div>
        </template>
    </Dialog>

    <Toast />
</template>

<style scoped>
@keyframes shake {
    0%,
    100% {
        transform: translateX(0);
    }
    10%,
    30%,
    50%,
    70%,
    90% {
        transform: translateX(-5px);
    }
    20%,
    40%,
    60%,
    80% {
        transform: translateX(5px);
    }
}

.shake-animation {
    animation: shake 0.6s ease-in-out;
}
</style>
