import React, { useState } from 'react';
import { useMutation, useQueryClient } from '@tanstack/react-query';
import { base44 } from '@/api/base44Client';
import { format } from 'date-fns';
import { Plus, Zap, BatteryCharging } from 'lucide-react';
import { Button } from '@/components/ui/button';
import { motion, AnimatePresence } from 'framer-motion';
import { toast } from 'sonner';
import { useNavigate } from 'react-router-dom';
import ActivityChip from '../components/log/ActivityChip';
import CreateActivityDialog from '../components/log/CreateActivityDialog';

const DRAIN_PRESETS = [
  { label: 'Small talk', points: 10 },
  { label: 'Work meeting', points: 20 },
  { label: 'Phone call', points: 15 },
  { label: 'Large gathering', points: 35 },
  { label: 'Networking event', points: 30 },
  { label: 'Explaining myself', points: 15 },
  { label: 'Unexpected visitors', points: 25 },
  { label: 'Group project', points: 20 },
];

const RECHARGE_PRESETS = [
  { label: 'Reading alone', points: 15 },
  { label: 'Nap', points: 25 },
  { label: 'Nature walk', points: 20 },
  { label: 'Gaming', points: 15 },
  { label: 'Hot shower', points: 10 },
  { label: 'Cancelled plans', points: 30 },
  { label: 'Cozy music', points: 10 },
  { label: 'Pet cuddles', points: 20 },
];

export default function Log() {
  const [selectedDrains, setSelectedDrains] = useState([]);
  const [selectedRecharges, setSelectedRecharges] = useState([]);
  const [drainOptions, setDrainOptions] = useState(DRAIN_PRESETS);
  const [rechargeOptions, setRechargeOptions] = useState(RECHARGE_PRESETS);
  const [dialogType, setDialogType] = useState(null);

  const queryClient = useQueryClient();
  const navigate = useNavigate();

  const logMutation = useMutation({
    mutationFn: (entries) => base44.entities.BatteryLog.bulkCreate(entries),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['battery-logs'] });
      setSelectedDrains([]);
      setSelectedRecharges([]);
      toast.success('Activities logged!');
      navigate('/');
    },
  });

  const toggleDrain = (label) => {
    setSelectedDrains(prev =>
      prev.includes(label) ? prev.filter(l => l !== label) : [...prev, label]
    );
  };

  const toggleRecharge = (label) => {
    setSelectedRecharges(prev =>
      prev.includes(label) ? prev.filter(l => l !== label) : [...prev, label]
    );
  };

  const handleLog = () => {
    const today = format(new Date(), 'yyyy-MM-dd');
    const entries = [
      ...selectedDrains.map(label => {
        const opt = drainOptions.find(o => o.label === label);
        return { label, type: 'drain', points: opt?.points || 10, date: today };
      }),
      ...selectedRecharges.map(label => {
        const opt = rechargeOptions.find(o => o.label === label);
        return { label, type: 'recharge', points: opt?.points || 10, date: today };
      }),
    ];
    if (entries.length === 0) return;
    logMutation.mutate(entries);
  };

  const handleCustomActivity = ({ label, points }) => {
    if (dialogType === 'drain') {
      setDrainOptions(prev => [...prev, { label, points }]);
    } else {
      setRechargeOptions(prev => [...prev, { label, points }]);
    }
  };

  const totalSelected = selectedDrains.length + selectedRecharges.length;

  return (
    <div className="px-5 pt-14 pb-6">
      <h1 className="text-2xl font-heading font-bold text-foreground mb-1">Log Activity</h1>
      <p className="text-xs text-muted-foreground font-body mb-8">
        What happened to your social battery today?
      </p>

      {/* Drains section */}
      <div className="mb-8">
        <div className="flex items-center gap-2 mb-4">
          <Zap className="w-4 h-4 text-destructive" />
          <h2 className="text-sm font-heading font-semibold text-foreground uppercase tracking-wide">
            Energy Drains
          </h2>
        </div>
        <div className="flex flex-wrap gap-2">
          {drainOptions.map((opt) => (
            <ActivityChip
              key={opt.label}
              label={opt.label}
              points={opt.points}
              type="drain"
              selected={selectedDrains.includes(opt.label)}
              onClick={() => toggleDrain(opt.label)}
            />
          ))}
          <button
            onClick={() => setDialogType('drain')}
            className="px-4 py-2.5 rounded-2xl text-sm font-medium font-body border border-dashed border-border text-muted-foreground hover:text-foreground hover:border-foreground/30 transition-colors flex items-center gap-1.5"
          >
            <Plus className="w-3.5 h-3.5" /> Custom
          </button>
        </div>
      </div>

      {/* Recharges section */}
      <div className="mb-10">
        <div className="flex items-center gap-2 mb-4">
          <BatteryCharging className="w-4 h-4 text-accent" />
          <h2 className="text-sm font-heading font-semibold text-foreground uppercase tracking-wide">
            Rechargers
          </h2>
        </div>
        <div className="flex flex-wrap gap-2">
          {rechargeOptions.map((opt) => (
            <ActivityChip
              key={opt.label}
              label={opt.label}
              points={opt.points}
              type="recharge"
              selected={selectedRecharges.includes(opt.label)}
              onClick={() => toggleRecharge(opt.label)}
            />
          ))}
          <button
            onClick={() => setDialogType('recharge')}
            className="px-4 py-2.5 rounded-2xl text-sm font-medium font-body border border-dashed border-border text-muted-foreground hover:text-foreground hover:border-foreground/30 transition-colors flex items-center gap-1.5"
          >
            <Plus className="w-3.5 h-3.5" /> Custom
          </button>
        </div>
      </div>

      {/* Log button */}
      <AnimatePresence>
        {totalSelected > 0 && (
          <motion.div
            initial={{ opacity: 0, y: 20 }}
            animate={{ opacity: 1, y: 0 }}
            exit={{ opacity: 0, y: 20 }}
            className="fixed bottom-28 left-0 right-0 px-5 max-w-lg mx-auto"
          >
            <Button
              onClick={handleLog}
              disabled={logMutation.isPending}
              className="w-full rounded-2xl h-14 text-base font-heading font-semibold shadow-lg"
            >
              {logMutation.isPending
                ? 'Logging...'
                : `Log ${totalSelected} ${totalSelected === 1 ? 'Activity' : 'Activities'}`}
            </Button>
          </motion.div>
        )}
      </AnimatePresence>

      <CreateActivityDialog
        open={dialogType !== null}
        onOpenChange={(open) => !open && setDialogType(null)}
        type={dialogType || 'drain'}
        onSave={handleCustomActivity}
      />
    </div>
  );
}
