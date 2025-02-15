<script setup>
import { ref, computed } from "vue";
import axios from "axios";

const updateTimeRegex = /\d{2}:\d{2}:\d{2} \d{2}-\d{2}-\d{4}/;
const violationCountRegex = /\d+/;
const detailViolationRegex = /\s*\(Xem chi tiết\)/;

const licensePlate = ref("");
const vehicleType = ref("");
const result = ref(null);
const loading = ref(false);
const domainName = window.location.hostname;

const isFormValid = computed(() => {
  return licensePlate.value.trim() !== "" && vehicleType.value !== "";
});

const formatResolutionText = (text) => {
  text = text.replace(/,(?=\d+\.)/g, "\n");
  text = text.replace(/,Địa chỉ:/g, "\nĐịa chỉ:");
  text = text.replace(/,Số điện thoại:/g, "\nSố điện thoại:");
  text = text.replace(/Số điện thoại:/g, "Số điện thoại liên hệ:");
  text = text.replace(/(Số điện thoại liên hệ:\s*)(\d{5})(\d{6})/g, "$1$2.$3");
  return text;
};

const parseHTMLResponse = (html) => {
  const parser = new DOMParser();
  const doc = parser.parseFromString(html, "text/html");

  if (html.includes("không có lỗi vi phạm")) {
    const updateTimeMatch = doc.querySelector("p")?.textContent.match(updateTimeRegex);
    return {
      violations: 0,
      licensePlate: licensePlate.value,
      updateTime: updateTimeMatch ? updateTimeMatch[0] : "",
      details: []
    };
  }

  const h3Text = doc.querySelector("h3")?.textContent || "";
  const paidText = doc.querySelector(".btn-success")?.textContent || "";
  const unpaidText = doc.querySelector(".btn-danger")?.textContent || "";
  const updateTimeMatch = doc.querySelector("p")?.textContent.match(updateTimeRegex);

  const tables = doc.querySelectorAll("table");
  const details = [];
  tables.forEach((table) => {
    const record = {};
    table.querySelectorAll("tr").forEach((row) => {
      const cells = row.querySelectorAll("td");
      if (cells.length >= 2) {
        const label = cells[0].textContent.trim().replace(":", "");
        let value = cells[1].textContent.trim();
        if (label === "Nơi giải quyết vụ việc") {
          value = formatResolutionText(value);
        }
        if (label === "Hành vi vi phạm") {
          value = value.replace(detailViolationRegex, "");
        }
        record[label] = value;
      }
    });
    if (Object.keys(record).length) {
      details.push(record);
    }
  });

  return {
    violations: parseInt(h3Text.match(violationCountRegex)?.[0] || "0", 10),
    paid: parseInt(paidText.match(violationCountRegex)?.[0] || "0", 10),
    unpaid: parseInt(unpaidText.match(violationCountRegex)?.[0] || "0", 10),
    updateTime: updateTimeMatch ? updateTimeMatch[0] : "",
    details,
  };
};

const handleSearch = async () => {
  try {
    loading.value = true;
    const { data } = await axios.get(
      `https://api.phatnguoi.vn/web/tra-cuu/${licensePlate.value}/${vehicleType.value}`
    );
    result.value = parseHTMLResponse(data);
  } catch (err) {
    console.error(err);
  } finally {
    loading.value = false;
  }
};

const resetForm = () => {
  licensePlate.value = "";
  vehicleType.value = "";
  result.value = null;
};
</script>

<template>
  <div class="container">
    <h1 class="title">Tra cứu phạt nguội</h1>

    <div class="card search-card">
      <div class="form-group">
        <input
          v-model="licensePlate"
          type="text"
          placeholder="Nhập biển số xe"
          class="input"
        />
      </div>
      <div class="form-group">
        <select v-model="vehicleType" class="select">
          <option disabled value="">Chọn loại xe</option>
          <option value="1">Ô tô</option>
          <option value="2">Xe máy</option>
          <option value="3">Xe máy điện</option>
        </select>
      </div>
      <button
        @click="handleSearch"
        :disabled="!isFormValid || loading"
        class="search-button"
      >
        <span v-if="loading">ĐANG KIỂM TRA...</span>
        <span v-else>KIỂM TRA PHẠT NGUỘI</span>
      </button>
    </div>

    <transition name="slide-fade">
      <div v-if="result" class="card result-card">
        <div v-if="result.violations > 0">
          <h3 class="result-title">
            Đã phát hiện {{ result.violations }} lỗi vi phạm
          </h3>
          <div class="status-badges">
            <span class="badge unpaid">{{ result.unpaid }} CHƯA XỬ PHẠT</span>
            <span class="badge paid">{{ result.paid }} ĐÃ XỬ PHẠT</span>
            <span class="badge search-again" @click="resetForm">
              <i class="fas fa-redo"></i>
              TRA CỨU TIẾP
            </span>
          </div>
          <div class="update-info">
            <p>
              Dữ liệu được cập nhật đến:
              <strong>{{ result.updateTime }}</strong> tại
              <span class="highlight">{{ domainName }}</span>
            </p>
          </div>
          <!-- Duyệt qua từng bảng vi phạm -->
          <div v-for="(record, index) in result.details" :key="index">
            <table class="violation-details">
              <tr v-for="(value, key) in record" :key="key">
                <td class="detail-label">{{ key }}:</td>
                <td class="detail-value">
                  <template v-if="key === 'Trạng thái'">
                    <span :class="value === 'CHƯA XỬ PHẠT' ? 'badge unpaid' : 'badge paid'">
                      {{ value }}
                    </span>
                  </template>
                  <template v-else-if="key === 'Biển kiểm soát'">
                    <strong>{{ value }}</strong>
                  </template>
                  <template v-else>
                    {{ value }}
                  </template>
                </td>
              </tr>
            </table>
            <!-- Khoảng cách giữa các bảng -->
            <hr v-if="index !== result.details.length - 1" />
          </div>
        </div>
        <div v-else class="no-violation">
          <div class="no-violation-icon">🎉</div>
          <p class="clean-record">KHÔNG TÌM THẤY VI PHẠM PHẠT NGUỘI!</p>
          <span class="update-time">
            Biển số <strong>{{ result.licensePlate }}</strong> | Dữ liệu được cập nhật đến:
            <strong>{{ result.updateTime }}</strong>
          </span>
          <div class="recommendation">
            <h4>Lưu ý quan trọng</h4>
            <p>
              Thông thường các lỗi phạt nguội (nếu có) sẽ có kết quả tra cứu từ 3 đến 15 ngày. 
              Bạn hãy thường xuyên kiểm tra phạt nguội để có kết quả chính xác nhất.
            </p>
            <button class="check-again-button" @click="handleSearch">
              <i class="fas fa-sync-alt"></i> KIỂM TRA LẠI
            </button>
          </div>
        </div>
        <div class="data-source">
          Nguồn: Cổng thông tin điện tử Cục Cảnh sát giao thông
        </div>
      </div>
    </transition>
  </div>
</template>

<style scoped>
:global(body) {
  background: linear-gradient(135deg, #f0f0f0, #ffffff);
  margin: 0;
  padding: 0;
  box-sizing: border-box;
  font-family: "Poppins", sans-serif;
}

.container {
  max-width: 800px;
  margin: 2rem auto;
  padding: 2rem;
  background: #ffffff;
  border-radius: 12px;
  box-shadow: 0 4px 15px rgba(0, 0, 0, 0.1);
  color: #333;
}

.title {
  text-align: center;
  margin-bottom: 2rem;
  font-size: 2.2rem;
  color: #007bff;
}

.card {
  background: #f7f7f7;
  padding: 1.8rem;
  border-radius: 12px;
  margin-bottom: 2rem;
  transition: transform 0.3s ease;
}

.search-card .form-group {
  margin-bottom: 1rem;
}

.input,
.select {
  width: 100%;
  box-sizing: border-box;
  padding: 0.75rem;
  border-radius: 8px;
  border: 1px solid #ccc;
  background: #ffffff;
  color: #333;
  transition: border-color 0.3s ease;
}

.input:focus,
.select:focus {
  border-color: #007bff;
  outline: none;
}

.search-button {
  width: 100%;
  padding: 0.75rem;
  background: #007bff;
  color: #ffffff;
  border: none;
  border-radius: 8px;
  cursor: pointer;
  transition: background 0.3s ease, transform 0.2s;
}
.search-button:disabled {
  background: #cccccc;
  cursor: not-allowed;
}
.search-button:hover:not(:disabled) {
  transform: scale(1.02);
}

.result-title {
  margin-bottom: 1rem;
  color: #007bff;
  text-align: center;
}

.status-badges {
  display: flex;
  justify-content: center;
  gap: 1rem;
  margin: 1.5rem 0;
}

.badge {
  padding: 0.5rem 1rem;
  border-radius: 20px;
  font-weight: bold;
  color: #fff;
}
.paid {
  background: #28a745;
}
.unpaid {
  background: #dc3545;
}
.search-again {
  background: #17a2b8;
  border: none;
  color: #fff;
  cursor: pointer;
  border-radius: 8px;
  padding: 0.5rem 1rem;
  transition: transform 0.2s;
}
.search-again:hover {
  transform: scale(1.03);
}

.update-info {
  text-align: center;
  margin-bottom: 1rem;
  font-size: 0.95rem;
}
.highlight {
  color: #007bff;
  font-weight: bold;
}

.violation-details {
  width: 100%;
  border-collapse: collapse;
  margin-top: 1rem;
}
.violation-details tr {
  border-bottom: 1px solid #ccc;
}
.detail-label {
  padding: 1rem;
  color: #007bff;
  width: 30%;
}
.detail-value {
  white-space: pre-line;
  padding: 1rem;
  color: #555;
  line-height: 1.5;
}

.no-violation {
  text-align: center;
  border: 2px solid #28a745;
  padding: 2rem;
  border-radius: 12px;
}
.no-violation-icon {
  font-size: 4rem;
  color: #28a745;
  margin-bottom: 1rem;
  animation: bounce 1s infinite;
}
@keyframes bounce {
  0%, 100% { transform: translateY(0); }
  50% { transform: translateY(-10px); }
}
.clean-record {
  font-size: 1.2rem;
  color: #28a745;
  margin-bottom: 1rem;
}
.update-time {
  font-size: 0.9rem;
  color: #666;
  margin-bottom: 1rem;
}

.recommendation {
  background: #e9ecef;
  padding: 1.5rem;
  border-radius: 12px;
  margin-top: 1rem;
  text-align: left;
}
.recommendation h4 {
  color: #007bff;
  margin-bottom: 1rem;
}
.recommendation p {
  color: #555;
  line-height: 1.5;
}

.check-again-button {
  background: #17a2b8;
  color: #fff;
  padding: 0.75rem 1.5rem;
  border: none;
  border-radius: 25px;
  cursor: pointer;
  transition: transform 0.2s ease;
  margin-top: 1rem;
}
.check-again-button:hover {
  transform: scale(1.05);
}

.data-source {
  text-align: center;
  margin-top: 1rem;
  color: #007bff;
  font-weight: bold;
}

@media (max-width: 1024px) {
  .container {
    margin: 1.5rem auto;
    padding: 1.5rem;
  }
  .title {
    font-size: 2rem;
  }
  .card {
    padding: 1.5rem;
  }
  .input,
  .select {
    padding: 0.65rem;
  }
  .search-button {
    padding: 0.65rem;
    font-size: 0.95rem;
  }
  .detail-label, .detail-value {
    padding: 0.75rem;
  }
}

@media (max-width: 768px) {
  .container {
    width: 90%;
    margin: 1rem auto;
    padding: 1rem;
  }
  .title {
    font-size: 1.8rem;
  }
  .card {
    padding: 1.2rem;
  }
  .input,
  .select {
    padding: 0.5rem;
    font-size: 0.9rem;
  }
  .search-button {
    padding: 0.5rem;
    font-size: 0.9rem;
  }
  .result-title {
    font-size: 1.5rem;
  }
  .detail-label, .detail-value {
    font-size: 0.9rem;
    padding: 0.5rem;
  }
  .status-badges {
    flex-direction: column;
    gap: 0.5rem;
    align-items: center;
  }
  .search-again {
    padding: 0.5rem;
    font-size: 0.9rem;
  }
  .update-info,
  .data-source {
    font-size: 0.85rem;
  }
}
  
.slide-fade-enter-active {
  transition: all 0.5s ease;
}
.slide-fade-enter-from {
  transform: translateY(10px);
  opacity: 0;
}
</style>

