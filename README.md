// App.js
import React, { useState, useEffect } from 'react';
import { NavigationContainer } from '@react-navigation/native';
import { createBottomTabNavigator } from '@react-navigation/bottom-tabs';
import { createNativeStackNavigator } from '@react-navigation/native-stack';
import AsyncStorage from '@react-native-async-storage/async-storage';
import * as FileSystem from 'react-native-fs';
import Share from 'react-native-share';
import { LineChart } from 'react-native-chart-kit';
import {
  View,
  Text,
  TextInput,
  TouchableOpacity,
  FlatList,
  StyleSheet,
  ScrollView,
  Alert,
  Dimensions,
  Modal
} from 'react-native';
import Icon from 'react-native-vector-icons/MaterialCommunityIcons';

const Stack = createNativeStackNavigator();
const Tab = createBottomTabNavigator();

// شاشة قائمة الموظفين
const EmployeesScreen = ({ navigation }) => {
  const [employees, setEmployees] = useState([]);
  const [newEmployee, setNewEmployee] = useState('');

  useEffect(() => {
    loadEmployees();
  }, []);

  const loadEmployees = async () => {
    try {
      const savedEmployees = await AsyncStorage.getItem('employees');
      if (savedEmployees) {
        setEmployees(JSON.parse(savedEmployees));
      }
    } catch (error) {
      Alert.alert('خطأ', 'حدث خطأ في تحميل البيانات');
    }
  };

  const saveEmployees = async (data) => {
    try {
      await AsyncStorage.setItem('employees', JSON.stringify(data));
    } catch (error) {
      Alert.alert('خطأ', 'حدث خطأ في حفظ البيانات');
    }
  };

  const addEmployee = () => {
    if (newEmployee.trim()) {
      const newEmployeeData = {
        id: Date.now().toString(),
        name: newEmployee,
        notes: [],
        averagePerformance: 0,
        department: '',
        position: ''
      };
      const updatedEmployees = [...employees, newEmployeeData];
      setEmployees(updatedEmployees);
      saveEmployees(updatedEmployees);
      setNewEmployee('');
    }
  };

  const deleteEmployee = (employeeId) => {
    Alert.alert(
      'تأكيد الحذف',
      'هل أنت متأكد من حذف هذا الموظف؟',
      [
        { text: 'إلغاء', style: 'cancel' },
        {
          text: 'حذف',
          style: 'destructive',
          onPress: async () => {
            const updatedEmployees = employees.filter(emp => emp.id !== employeeId);
            setEmployees(updatedEmployees);
            await saveEmployees(updatedEmployees);
          }
        }
      ]
    );
  };

  return (
    <View style={styles.container}>
      <View style={styles.addEmployeeContainer}>
        <TextInput
          style={styles.input}
          value={newEmployee}
          onChangeText={setNewEmployee}
          placeholder="اسم الموظف الجديد"
          placeholderTextColor="#666"
        />
        <TouchableOpacity style={styles.addButton} onPress={addEmployee}>
          <Icon name="plus" size={24} color="white" />
        </TouchableOpacity>
      </View>

      <FlatList
        data={employees}
        keyExtractor={(item) => item.id}
        renderItem={({ item }) => (
          <View style={styles.employeeCard}>
            <TouchableOpacity
              style={styles.employeeInfo}
              onPress={() => navigation.navigate('EmployeeDetails', { employee: item })}
            >
              <Text style={styles.employeeName}>{item.name}</Text>
              <Text style={styles.performanceText}>
                متوسط الأداء: {item.averagePerformance.toFixed(1)} / 5
              </Text>
            </TouchableOpacity>
            <TouchableOpacity
              style={styles.deleteButton}
              onPress={() => deleteEmployee(item.id)}
            >
              <Icon name="delete" size={24} color="#ff4444" />
            </TouchableOpacity>
          </View>
        )}
      />
    </View>
  );
};

// شاشة تفاصيل الموظف
const EmployeeDetailsScreen = ({ route, navigation }) => {
  const { employee } = route.params;
  const [note, setNote] = useState('');
  const [rating, setRating] = useState(5);
  const [showRatingModal, setShowRatingModal] = useState(false);

  const addNote = async () => {
    if (note.trim()) {
      try {
        const savedEmployees = await AsyncStorage.getItem('employees');
        if (savedEmployees) {
          const employees = JSON.parse(savedEmployees);
          const updatedEmployees = employees.map(emp => {
            if (emp.id === employee.id) {
              const newNote = {
                id: Date.now().toString(),
                content: note,
                date: new Date().toISOString().split('T')[0],
                rating: rating
              };
              const updatedNotes = [...emp.notes, newNote];
              return {
                ...emp,
                notes: updatedNotes,
                averagePerformance: updatedNotes.reduce((acc, n) => acc + n.rating, 0) / updatedNotes.length
              };
            }
            return emp;
          });
          await AsyncStorage.setItem('employees', JSON.stringify(updatedEmployees));
          setNote('');
          setRating(5);
          navigation.setParams({ employee: updatedEmployees.find(emp => emp.id === employee.id) });
        }
      } catch (error) {
        Alert.alert('خطأ', 'حدث خطأ في حفظ الملاحظة');
      }
    }
  };

  const RatingModal = () => (
    <Modal
      transparent={true}
      visible={showRatingModal}
      onRequestClose={() => setShowRatingModal(false)}
    >
      <View style={styles.modalContainer}>
        <View style={styles.modalContent}>
          <Text style={styles.modalTitle}>تقييم الأداء</Text>
          <View style={styles.ratingContainer}>
            {[1, 2, 3, 4, 5].map((value) => (
              <TouchableOpacity
                key={value}
                style={[
                  styles.ratingButton,
                  rating === value && styles.selectedRating
                ]}
                onPress={() => setRating(value)}
              >
                <Text style={[
                  styles.ratingButtonText,
                  rating === value && styles.selectedRatingText
                ]}>
                  {value}
                </Text>
              </TouchableOpacity>
            ))}
          </View>
          <TouchableOpacity
            style={styles.modalButton}
            onPress={() => setShowRatingModal(false)}
          >
            <Text style={styles.modalButtonText}>تم</Text>
          </TouchableOpacity>
        </View>
      </View>
    </Modal>
  );

  return (
    <ScrollView style={styles.container}>
      <RatingModal />
      
      <View style={styles.noteInputContainer}>
        <TextInput
          style={styles.noteInput}
          value={note}
          onChangeText={setNote}
          placeholder="أضف ملاحظة جديدة..."
          multiline
          placeholderTextColor="#666"
        />
        <View style={styles.noteActions}>
          <TouchableOpacity
            style={styles.ratingSelector}
            onPress={() => setShowRatingModal(true)}
          >
            <Text style={styles.ratingText}>التقييم: {rating}/5</Text>
            <Icon name="star" size={20} color="#FFD700" />
          </TouchableOpacity>
          <TouchableOpacity style={styles.addButton} onPress={addNote}>
            <Text style={styles.buttonText}>إضافة</Text>
          </TouchableOpacity>
        </View>
      </View>

      <FlatList
        data={employee.notes.slice().reverse()}
        keyExtractor={(item) => item.id}
        renderItem={({ item }) => (
          <View style={styles.noteCard}>
            <View style={styles.noteHeader}>
              <Text style={styles.dateText}>{item.date}</Text>
              <View style={styles.ratingBadge}>
                <Text style={styles.ratingBadgeText}>{item.rating}/5</Text>
                <Icon name="star" size={16} color="#FFD700" />
              </View>
            </View>
            <Text style={styles.noteContent}>{item.content}</Text>
          </View>
        )}
      />
    </ScrollView>
  );
};

// شاشة التقارير
const ReportsScreen = () => {
  const [employees, setEmployees] = useState([]);
  const [selectedMonth, setSelectedMonth] = useState(new Date().toISOString().slice(0, 7));
  
  useEffect(() => {
    loadEmployees();
  }, []);

  const loadEmployees = async () => {
    try {
      const savedEmployees = await AsyncStorage.getItem('employees');
      if (savedEmployees) {
        setEmployees(JSON.parse(savedEmployees));
      }
    } catch (error) {
      Alert.alert('خطأ', 'حدث خطأ في تحميل البيانات');
    }
  };

  const getMonthlyStats = (employee) => {
    const monthNotes = employee.notes.filter(note => note.date.startsWith(selectedMonth));
    const totalNotes = monthNotes.length;
    const averageRating = totalNotes > 0 
      ? monthNotes.reduce((acc, note) => acc + note.rating, 0) / totalNotes 
      : 0;
    const daysWorked = new Set(monthNotes.map(note => note.date)).size;
    
    return {
      totalNotes,
      averageRating,
      daysWorked,
      performanceData: monthNotes.map(note => ({
        date: note.date,
        rating: note.rating
      }))
    };
  };

  const exportReport = async () => {
    try {
      const reportContent = employees.map(employee => {
        const stats = getMonthlyStats(employee);
        return `الموظف: ${employee.name}\nعدد الملاحظات: ${stats.totalNotes}\nعدد الأيام العاملة: ${stats.daysWorked}\nمتوسط التقييم: ${stats.averageRating.toFixed(2)}\n`;
      }).join('\n\n');
      
      const path = `${FileSystem.DocumentDirectoryPath}/report.txt`;
      await FileSystem.writeFile(path, reportContent);
      await Share.open({ url: `file://${path}` });
    } catch (error) {
      Alert.alert('خطأ', 'حدث خطأ أثناء تصدير التقرير');
    }
  };

  return (
    <View style={styles.container}>
      <Text style={styles.reportTitle}>تقارير الأداء الشهرية</Text>
      <View style={styles.chartContainer}>
        <LineChart
          data={{
            labels: ['يوم 1', 'يوم 2', 'يوم 3', 'يوم 4'],
            datasets: [
              {
                data: [3, 4, 2, 5],
                strokeWidth: 2
              }
            ]
          }}
          width={Dimensions.get('window').width - 30}
          height={220}
          chartConfig={{
            backgroundColor: '#fff',
            backgroundGradientFrom: '#fff',
            backgroundGradientTo: '#fff',
            decimalPlaces: 2,
            color: (opacity = 1) => `rgba(0, 0, 255, ${opacity})`,
            labelColor: (opacity = 1) => `rgba(0, 0, 0, ${opacity})`
          }}
          style={styles.chart}
        />
      </View>
      <TouchableOpacity style={styles.exportButton} onPress={exportReport}>
        <Text style={styles.buttonText}>تصدير التقرير</Text>
      </TouchableOpacity>
    </View>
  );
};

// الإعدادات والتنسيقات
const styles = StyleSheet.create({
  container: {
    flex: 1,
    backgroundColor: '#fff',
    padding: 10,
  },
  addEmployeeContainer: {
    flexDirection: 'row',
    marginBottom: 20,
    alignItems: 'center'
  },
  input: {
    flex: 1,
    borderWidth: 1,
    borderColor: '#ccc',
    borderRadius: 5,
    padding: 10,
    marginRight: 10,
  },
  addButton: {
    backgroundColor: '#2e8b57',
    padding: 10,
    borderRadius: 5,
  },
  employeeCard: {
    flexDirection: 'row',
    justifyContent: 'space-between',
    alignItems: 'center',
    padding: 10,
    borderBottomWidth: 1,
    borderColor: '#eee',
  },
  employeeInfo: {
    flex: 1,
  },
  employeeName: {
    fontSize: 18,
    fontWeight: 'bold',
  },
  performanceText: {
    color: '#777',
  },
  deleteButton: {
    marginLeft: 10,
  },
  buttonText: {
    color: 'white',
    fontSize: 16,
  },
  modalContainer: {
    flex: 1,
    justifyContent: 'center',
    alignItems: 'center',
    backgroundColor: 'rgba(0, 0, 0, 0.5)',
  },
  modalContent: {
    backgroundColor: 'white',
    padding: 20,
    borderRadius: 5,
    width: '80%',
  },
  modalTitle: {
    fontSize: 18,
    fontWeight: 'bold',
    marginBottom: 10,
  },
  ratingContainer: {
    flexDirection: 'row',
    justifyContent: 'space-evenly',
    marginBottom: 20,
  },
  ratingButton: {
    padding: 10,
    backgroundColor: '#f0f0f0',
    borderRadius: 5,
  },
  selectedRating: {
    backgroundColor: '#FFD700',
  },
  ratingButtonText: {
    fontSize: 18,
  },
  selectedRatingText: {
    color: 'white',
  },
  modalButton: {
    backgroundColor: '#2e8b57',
    padding: 10,
    borderRadius: 5,
    alignItems: 'center',
  },
  modalButtonText: {
    color: 'white',
    fontSize: 16,
  },
  noteInputContainer: {
    marginBottom: 20,
  },
  noteInput: {
    borderWidth: 1,
    borderColor: '#ccc',
    padding: 10,
    borderRadius: 5,
    marginBottom: 10,
  },
  noteActions: {
    flexDirection: 'row',
    justifyContent: 'space-between',
  },
  ratingSelector: {
    flexDirection: 'row',
    alignItems: 'center',
  },
  ratingText: {
    marginRight: 10,
    fontSize: 16,
  },
  addButton: {
    backgroundColor: '#2e8b57',
    padding: 10,
    borderRadius: 5,
  },
  noteCard: {
    marginBottom: 15,
    padding: 10,
    borderWidth: 1,
    borderColor: '#ccc',
    borderRadius: 5,
  },
  noteHeader: {
    flexDirection: 'row',
    justifyContent: 'space-between',
    marginBottom: 5,
  },
  dateText: {
    fontSize: 14,
    color: '#777',
  },
  ratingBadge: {
    flexDirection: 'row',
    alignItems: 'center',
  },
  ratingBadgeText: {
    fontSize: 14,
    color: '#FFD700',
  },
  noteContent: {
    fontSize: 16,
    marginTop: 5,
  },
  reportTitle: {
    fontSize: 20,
    fontWeight: 'bold',
    marginBottom: 20,
  },
  chartContainer: {
    marginBottom: 20,
  },
  chart: {
    borderRadius: 5,
  },
  exportButton: {
    backgroundColor: '#2e8b57',
    padding: 10,
    borderRadius: 5,
    alignItems: 'center',
  },
});

export default App;
